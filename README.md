# Step 1: Power Button → UEFI/BIOS Init (CPU + GPU + Firmware)

---

## 1. Power Flow (Electricity stage)
- You press the power button → motherboard short-circuits the signal (PS_ON# line).
- Power Supply Unit (PSU) sends “Power Good” signal → ensures voltages (3.3V, 5V, 12V) are stable.
- Once signal is steady → CPU reset pin is released.
- CPU wakes up for the first time.

**Note:** Until now = pure electrical. No software, no Linux, no `/boot`.

---

## 2. CPU Reset Vector
- CPU has no OS, no filesystem, no RAM content—it’s blank.
- CPU designers hardwired: after reset, the Instruction Pointer (IP) must start from a fixed address.
- For x86, usually `0xFFFFFFF0` (near top of 4GB).
- That address belongs to the UEFI/BIOS chip (a flash ROM soldered on motherboard).
- CPU fetches first instructions from there.

**This is the firmware stage: CPU asking “What should I do first?”**

---

## 3. UEFI Firmware (ROM on motherboard)

UEFI (or legacy BIOS if old system) does the following:

1. **Initialize CPU**
   - Loads CPU microcode patches (from Intel/AMD).
   - Switches from “real mode” to a richer CPU mode if UEFI.
   - Prepares interrupt vectors.

2. **Check Memory (POST)**
   - Basic memory test.
   - Builds a memory map (which regions are RAM, ROM, reserved).

3. **Detect buses**
   - PCI/PCIe enumeration → scans devices connected (GPU, network card, storage controllers).
   - Each device responds with IDs (Vendor ID + Device ID).
   - Stores info in tables for later.

**At this stage, UEFI = conductor waking up all orchestra players.**

---

## 4. GPU Initialization (Firmware inside GPU)
- Every modern GPU has its own small ROM chip called VBIOS (legacy) or UEFI GOP driver (modern).
- When UEFI scans PCIe devices, it finds GPU.
- UEFI says: “Hey GPU, give me your firmware routine.”
- GPU’s ROM provides:
  - Framebuffer driver (simple drawing to video memory).
  - EDID handshake (monitor tells supported resolutions, refresh rates).
  - Optionally UEFI menus (some fancy motherboards draw graphical BIOS setup).

**This is why you see:**
- Motherboard logo, vendor splash screen.
- BIOS/UEFI configuration menus.
- Even before Linux kernel is touched.

---

## 5. Output Stage (CPU ↔ GPU ↔ Monitor)
- CPU tells GPU: “Please draw this framebuffer (logo, text).”
- GPU uses its onboard VRAM (or system RAM if integrated GPU).
- GPU pushes signal via HDMI/DP cable.
- Monitor’s EDID chip says “I support 1920x1080 @ 60Hz” → GPU aligns output.

**At this stage → GPU is awake but not doing 3D acceleration.  
Only basic text/bitmap drawing via firmware.**

---

## 6. Linux Filesystem Context (Not Yet Born)
- No `/boot`, `/etc`, `/dev`.
- Only firmware world: CPU + GPU using ROM chips.
- RAM is being tested, but not filled.
- Storage (SSD/HDD) not touched yet.

**Analogy:** You just woke up in the morning, only wearing your underwear.  
Clothes (`/boot` etc.) still in cupboard (SSD).

---

## Key Microscopic Notes
- Both CPU and GPU have their own ROMs.
- CPU ROM = UEFI firmware (full system init).
- GPU ROM = VBIOS/UEFI GOP (basic graphics init).
- UEFI doesn’t yet touch Linux files. It only preps environment so bootloader can later read `/boot`.
- At this stage: if something fails, you hear beep codes or see error lights.

---

# Step 2: Bootloader Discovery (UEFI → /boot)

---

## 1. CPU State (after firmware)
- CPU is now awake + stable, executing UEFI firmware code.
- UEFI built a system map:
  - What RAM is usable.
  - What PCI devices exist.
  - Where storage controllers (SATA/NVMe) are.
- CPU is still executing instructions inside firmware’s protected memory, not touching Linux yet.

---

## 2. UEFI Locates Bootloader
- UEFI has a Boot Manager (stored in NVRAM, persistent memory on motherboard).
- This Boot Manager has boot entries (like: “boot from SSD partition X”).
- It looks for:
  - EFI System Partition (ESP) = a small FAT32 partition on SSD.
  - Inside `/EFI/` directory → each OS has its bootloader:
    - `/EFI/ubuntu/grubx64.efi`
    - `/EFI/Microsoft/Boot/bootmgfw.efi`

**This is the moment UEFI decides: “Linux GRUB? Or Windows Boot Manager?”**

---

## 3. GPU Role
- GPU still showing firmware framebuffer → splash logo or text “Press F2 for setup”.
- Still very primitive (no acceleration, no Wayland).
- UEFI draws directly into GPU framebuffer.
- No Linux drivers yet.

---

## 4. Bootloader Stage Begins
- Once UEFI selects GRUB (Linux bootloader), it loads `GRUB.efi` from the ESP into RAM.
- CPU now executes GRUB’s code.
- At this point → we are officially out of firmware land, into OS boot land.

---

## 5. GRUB Actions
GRUB’s job is:

1. **Draw splash menu (choose kernel, recovery mode, other OS).**
   - Uses its own minimal drivers → reads GPU framebuffer (still firmware-level).

2. **Locate `/boot` partition.**
   - GRUB can read ext4/xfs/etc. (has its own mini filesystem drivers).
   - `/boot` contains:
     - Kernel image (e.g. `vmlinuz-6.5.0`)
     - initramfs (e.g. `initrd.img-6.5.0`)
     - GRUB config (`grub.cfg`).

3. **Load kernel + initramfs into RAM.**

4. **Pass control to kernel.**

**GRUB = the bridge between firmware and Linux kernel.**

---

## 6. Linux Filesystem Context
- First time storage comes into play!
- `/boot` partition = crucial:
  - Contains kernel + initramfs.
- If `/boot` corrupt → system won’t start.
- Still no `/etc`, `/dev`, `/var` yet (those live on root filesystem `/`).
- Only `/boot` touched.

---

## 7. initramfs Role
- GRUB doesn’t just load kernel → it also loads initramfs (Initial RAM Filesystem).
- Why? Because:
  - Kernel is compressed + minimal.
  - Needs helper tools (drivers, mount helpers) → initramfs provides them.
- Example: If root filesystem is on LVM, RAID, or encrypted → initramfs has those drivers.

**initramfs = “early toolbox” in RAM.**

---

## 8. Transition CPU → Kernel
- GRUB prepares a kernel command line (parameters). 

Example: 
```bash
root=/dev/nvme0n1p2
```

- Tells kernel:
  - Where root filesystem is.
  - Mount read-only first.
  - Show splash screen quietly.

- CPU jumps into kernel entry point (the very first assembly line of Linux kernel).

**Firmware is now 100% out of the picture.**

---

## 9. GPU State
- Still showing splash menu or loading bar (plymouth).
- Framebuffer still handled by VBIOS or basic kernel mode driver.
- KMS not yet active (that happens in Step 3).


---


# Step 3: Kernel Initialization

---

## 1. CPU Enters the Kernel
- GRUB just jumped into the kernel’s entry point (tiny assembly stub in `arch/x86/boot/head.S` for x86).
- Kernel image was compressed → first thing CPU does = uncompress it into RAM.
- Now the real Linux kernel binary is alive in memory.

👉 Think of it as unzipping a giant toolbox from `/boot/vmlinuz` into RAM.

---

## 2. Early Kernel Setup
- Kernel initializes the CPU state:
  - Switches from **real mode → protected mode → long mode (64-bit)**.
  - Enables **memory paging**.
  - Sets up **temporary page tables**.
  - Initializes very basic **interrupt handling** (keyboard press, timers, etc.).
- So CPU now has a safe “playground” where Linux runs independently of firmware.

---

## 3. GPU Role (still basic)
- GPU still in framebuffer mode (set by BIOS/UEFI firmware).
- Kernel hasn’t yet activated its full graphics drivers.
- At this point → you might see only text logs scrolling (`dmesg` messages).

👉 The pretty graphics won’t happen until **KMS kicks in**.

---

## 4. initramfs Gets Mounted
- Kernel mounts the initramfs that GRUB loaded into RAM.
- `/` (root) at this moment = temporary filesystem inside RAM.
- initramfs contains:
  - Drivers (for SATA, NVMe, USB, etc.).
  - Tools like `udev` to detect devices.
  - Mount helpers (so we can reach the real root filesystem).

**Example:**  
If root FS is on an NVMe drive → kernel loads NVMe driver from initramfs → then can see `/dev/nvme0n1p2`.

---

## 5. Device Discovery (udev starts working)
- Kernel probes hardware buses (PCI, USB).
- Detects:
  - CPU cores
  - RAM
  - Storage
  - Network cards
  - GPU
- For each device:
  - A corresponding file is created under `/dev` (e.g. `/dev/sda`, `/dev/fb0`).
  - `udev` manages these dynamically.

👉 This is the first time `/dev` directory fills up.

---

## 6. Root Filesystem Switch
- Once kernel + initramfs know how to talk to the disk:
  - Kernel mounts the real root filesystem (`/`) from SSD/HDD.
  - Example: `/dev/nvme0n1p2` gets mounted at `/`.
- Then kernel does a **pivot_root**:
  - Replaces temporary initramfs `/` with the real one.
- After this pivot:
  - `/etc`, `/var`, `/usr`, `/home` are now available.
  - Kernel can run real user programs.

---

## 7. Start of Kernel Mode Drivers
- Kernel now starts loading kernel modules (`.ko` files) from `/lib/modules/`.
- These are extra drivers not built into the kernel itself.
- Example:
  - `i915.ko` → Intel GPU driver
  - `amdgpu.ko` → AMD GPU driver
  - `nvme.ko` → NVMe SSD driver

👉 This is where the **real GPU driver loads** for the first time.

---

## 8. Enter Kernel Mode Setting (KMS)
- Kernel hands display control to KMS (**Kernel Mode Setting**).
- KMS job:
  - Set resolution (from **EDID** of monitor).
  - Set refresh rate (60Hz, 144Hz, etc.).
  - Allocate framebuffers in GPU memory (VRAM).
- At this stage → you see the **first resolution change** (screen flicker, splash screen changes size).

👉 GPU is now officially “owned” by the kernel, not firmware anymore.

---

## 9. Spawn PID 1 (init/systemd)
- Once hardware + filesystems are stable → kernel runs `/sbin/init` (or `/lib/systemd/systemd`).
- This is **PID 1**, the parent of all processes.
- Kernel’s job is done → it hands over system management to **init/systemd**.

👉 From here on → it’s **userland territory** (services, login, GUI).

---


# Step 4: init/systemd (PID 1) → Userland Boot

---

## 1. Kernel Hands Control to PID 1
- Kernel just finished mounting the real root FS.
- Now it looks for the first program to run: `/sbin/init` (or `/lib/systemd/systemd`).
- This becomes **PID 1** (the first user-space process).
- From now on:
  - Kernel only works in the background (drivers, syscalls, process scheduler).
  - System management = `init/systemd`.

👉 Kernel = the **engine**, `systemd/init` = the **driver**.

---

## 2. systemd Startup
- `systemd` replaces the old SysVinit runlevels with **targets**.
- Reads configs from:
  - `/etc/systemd/system/`
  - `/usr/lib/systemd/system/`
- Starts services **in parallel** (not one-by-one).
  - Example: networking + logging + sound start at the same time.
- Uses **cgroups** to manage resources (CPU/mem limits for each service).

**Example:**  
Your `sshd` service (remote login) and `NetworkManager` start together.

---

## 3. Decide Target (CLI or GUI?)
- `systemd` checks the `default.target` symlink:
  - `graphical.target` → boot into GUI (GDM login).
  - `multi-user.target` → boot into CLI (server mode).
- Target is decided by distro defaults:
  - Ubuntu Desktop = GUI
  - CentOS Server = CLI
- You can override with:

```bash
  sudo systemctl set-default multi-user.target
  sudo systemctl set-default graphical.target
```
👉 This is the **exact moment** your system decides CLI vs GUI.

---

## 4. Start Display Manager (if GUI target)
- If `graphical.target` is set → `systemd` starts the **Display Manager** (e.g. GDM for GNOME).
- Display Manager responsibilities:
  - Starts **Xorg** or **Wayland compositor** (the graphics system).
  - Draws the login screen (greeter).
  - Authenticates user via `/etc/passwd` + `/etc/shadow`.

👉 Until now, GPU showed only kernel framebuffer (KMS).  
Now → GPU is driven by **Xorg/Wayland via DRM** → real desktop graphics begin.

---

## 5. Start CLI Getties (if CLI target)
- If `multi-user.target` is set:
  - `systemd` spawns `getty` processes on TTYs (text consoles).
  - These are the “login prompts” you see with `CTRL+ALT+F2`.
  - Default shell = `/bin/bash`.

👉 At this point, you type username + password → system starts your shell.

---

## 6. Filesystem Role
- `systemd` uses configs from `/etc/`:
  - `/etc/fstab` → which filesystems to mount.
  - `/etc/systemd/system/` → custom service overrides.
- `/var/` starts filling with logs:
  - `/var/log/journal/` = systemd-journald logs.
- `/dev/` already populated:
  - Now daemons start using devices (e.g. sound server using `/dev/snd`).

---

## 7. GPU Role at This Point
- GPU now under **Wayland/Xorg + compositor**.
- KMS/DRM still handle:
  - Mode-setting (resolution, refresh rate).
- GPU VRAM is used for:
  - **Framebuffers** (screen images).
  - **Textures** (icons, windows).

👉 User now sees:
- GUI login screen (if graphical.target).
- CLI login prompt (if multi-user.target).

---

# Step 5: User Login → Session Spawn

---

## 1. User Enters Credentials
- **CLI case (getty):**
  - Prompt: `login:`
  - Username checked in `/etc/passwd`.
  - Password checked in `/etc/shadow` (hashed + compared).
  - If valid → dropped into a shell (`/bin/bash` by default).

- **GUI case (GDM, LightDM, SDDM):**
  - Graphical greeter loads.
  - Username + password entered.
  - Display Manager authenticates using **PAM** (Pluggable Authentication Modules).
  - If valid → graphical session (GNOME, KDE, XFCE, etc.) starts.

👉 In both cases: `/etc/passwd` + PAM stack = **the gatekeeper**.

---

## 2. Session Start (systemd –user)
- After successful login:
  - `systemd` starts a **user manager instance** → `systemd --user`.
  - This manages **per-user services**:
    - Bluetooth tray, notifications, GNOME keyring, etc.
  - A **cgroup** is created for the user:
    - Controls CPU/memory usage.
    - Keeps user processes isolated from others.

👉 Every process you launch now belongs inside your **user session cgroup**.

---

## 3. Environment Setup
- **Shell session:**
  - Reads `/etc/profile`, `~/.bash_profile`, `~/.bashrc`.
  - Sets PATH, aliases, environment variables.

- **GUI session:**
  - Reads `/etc/X11/xinit/xinitrc` (if using X11).
  - Reads `/etc/xdg/` configs (global GNOME/KDE defaults).
  - Reads user configs in `~/.config/` (themes, keybindings, extensions).

👉 Filesystem now heavily involved:
- `/etc` → global configs.
- `/home/username` → personal configs (dotfiles).

---

## 4. GPU Role Now
- **GUI case:**
  - Display Manager hands control to `gnome-session` (or KDE’s session manager).
  - `gnome-session` starts **Mutter** (compositor).
  - Mutter talks to GPU via **Wayland/DRM** (or Xorg/DRM if fallback).
  - GPU renders windows, wallpaper, animations in VRAM using shaders.

- **CLI case:**
  - GPU stays in **text framebuffer** mode (KMS).
  - No compositor — just raw text display.

👉 This is the **turning point**: GPU fully takes charge of drawing the UI.

---

## 5. Daemons & Services Spawn
- After login, personal daemons start:
  - `dbus-daemon` → message bus for apps.
  - `pulseaudio` / `pipewire` → sound server.
  - `gvfs` → virtual FS for USB drives, network shares.
  - Tray apps → volume, WiFi, battery, clipboard managers.

👉 These run as **user processes**, not global system processes.

---

## 6. Final Result
- **CLI user:**
  - At a shell prompt: `jill@machine:~$`
  - CPU: running bash, waiting for commands.
  - GPU: only text framebuffer.
  - FS: `/home/jill/` is the playground.

- **GUI user:**
  - At GNOME/KDE/XFCE desktop (panels, icons, windows).
  - CPU: running many user processes.
  - GPU: rendering UI with compositing.
  - FS: 
    - `/usr/share/` → desktop assets.
    - `/home/jill/.config` → personal settings.

---

# Step 6: Desktop Running (Steady State)

---

## 1. Processes & CPU
- **System services (daemons):**
  - `systemd` (PID 1, master).
  - `journald` (logging).
  - `NetworkManager`, `sshd`, `cups` (printing), etc.
- **User session services:**
  - `systemd --user` (user session manager).
  - `dbus-daemon` (message bus).
  - `pipewire` / `pulseaudio` (sound).
  - Tray daemons (battery, Bluetooth, notifications).
- **Applications:**
  - Firefox, VSCode, terminal, etc.

👉 The **kernel scheduler** balances CPU time between system services and user apps.

---

## 2. GPU & Compositing
- GPU is now **fully owned by compositor** (Mutter for GNOME, KWin for KDE).
- Every window:
  - Drawn off-screen into textures in VRAM.
  - Compositor blends textures into a single image.
- Final image → sent via DRM/KMS → monitor.
- Effects (animations, shadows, transparency) = GPU shaders running in real time.

👉 Example: dragging a window = GPU redraws new frames 60–144 times per second.

---

## 3. Filesystem Roles
Multiple directories constantly in use:
- `/home/username` → user files, configs, downloads.
- `/usr/bin`, `/usr/lib` → apps and libraries currently running.
- `/etc` → global configs (DNS: `/etc/resolv.conf`, hosts: `/etc/hosts`).
- `/var/log` → constantly updated logs (`journalctl`, system logs).
- `/dev` → device files actively accessed (`/dev/snd/*` for audio, `/dev/dri/*` for GPU).
- `/proc`, `/sys` → kernel runtime data (used by tools like `top`, `free`).

👉 FS = the live map the kernel + apps consult constantly.

---

## 4. User Interactions
- Input events → go through `/dev/input`:
  - Mouse → `/dev/input/mice`.
  - Keyboard → `/dev/input/eventX`.
- Kernel delivers events → Wayland/Xorg → compositor → app window.
- App updates its state → GPU redraws pixels → monitor refresh.

👉 This cycle happens thousands of times per second.

---

## 5. Background Work
Even when idle:
- CPU handles hardware interrupts (network packets, disk writes).
- Kernel flushes memory pages → disk (writeback).
- Cron/systemd-timers run jobs (backups, indexing).
- Package managers may fetch updates silently.

---

## 6. Security Layer
- **PAM** → session security.
- **SELinux/AppArmor** → restricts process permissions.
- **cgroups + namespaces** → isolate apps and services.
- Used in **containers** (Docker, Podman, etc.).

👉 Prevents one rogue app from crashing or stealing the whole system.

---

## 7. What You See
- **CLI user:**
  - Stable shell prompt.
  - GPU → text framebuffer only.
  - CPU → idle until commands given.
- **GUI user:**
  - GNOME/KDE desktop with windows, icons, and tray.
  - GPU → constant compositing.
  - CPU → multitasking apps + background daemons.

👉 At this stage: Linux is **fully interactive** → multitasking with kernel invisibly managing resources.

---

✅ End of boot → steady state journey:  
**Power switch → firmware → bootloader → kernel → init/systemd → login → desktop running**




