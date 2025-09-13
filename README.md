# Linux Boot to Desktop: Deep Dive

This document explains, step by step, the complete journey from pressing the **power button** to reaching the **steady state Linux desktop**.  
It covers **CPU**, **GPU**, and the **Linux filesystem** involved at every stage.

---

## Step 1: Power Button → UEFI/BIOS Init (CPU + GPU + Firmware)

### 1. Power Flow (Electrical stage)
- Pressing power → motherboard short-circuits the PS_ON# line.
- Power Supply Unit (PSU) sends *Power Good* signal → ensures voltages (3.3V, 5V, 12V) are stable.
- Once steady → CPU reset pin released.
- CPU wakes up for the first time.

**Note:** Until now, this is **purely electrical**. No software, no Linux, no `/boot`.

---

### 2. CPU Reset Vector
- CPU has no OS, no filesystem, no RAM content.
- Hardwired start: Instruction Pointer (IP) = `0xFFFFFFF0` (x86).
- Address belongs to UEFI/BIOS chip (flash ROM).
- CPU fetches first instructions from there.

---

### 3. UEFI Firmware (ROM on motherboard)
UEFI does:
1. **Initialize CPU**
   - Loads microcode patches (Intel/AMD).
   - Switches CPU modes.
   - Prepares interrupt vectors.
2. **Check Memory (POST)**
   - Tests RAM.
   - Builds memory map.
3. **Detect Buses**
   - PCI/PCIe enumeration → scans GPU, network, storage.
   - Devices respond with Vendor ID + Device ID.
   - Stores info in tables.

---

### 4. GPU Initialization (Firmware inside GPU)
- Every GPU has its own ROM: **VBIOS** (legacy) or **UEFI GOP driver**.
- UEFI finds GPU → asks for its firmware routine.
- GPU firmware provides:
  - Framebuffer driver (basic drawing).
  - EDID handshake (monitor capabilities).
  - Optionally UEFI menus.

**Result:**  
You see motherboard logo, BIOS/UEFI setup menus → all *before* Linux.

---

### 5. Output Stage (CPU ↔ GPU ↔ Monitor)
- CPU tells GPU: draw framebuffer (logo, text).
- GPU uses VRAM/system RAM.
- GPU outputs via HDMI/DP cable.
- Monitor EDID: “I support 1920x1080@60Hz” → GPU aligns.

---

### 6. Linux Filesystem Context
- At this stage → **no `/boot`, `/etc`, `/dev`**.
- Only firmware in ROM + RAM being tested.
- Storage not touched.

---

### Key Microscopic Notes
- CPU ROM = UEFI firmware.
- GPU ROM = VBIOS/UEFI GOP.
- UEFI does **not** touch Linux yet, only prepares environment.
- Errors here → beep codes or lights.

---

## Step 2: Bootloader Discovery (UEFI → /boot)

### 1. CPU State
- CPU runs UEFI firmware code.
- UEFI map:
  - RAM layout.
  - PCI devices.
  - Storage controllers.
- Still inside firmware space.

---

### 2. UEFI Locates Bootloader
- Boot Manager in NVRAM stores entries.
- Looks for **EFI System Partition (ESP)**.
- Inside `/EFI/` → bootloaders:
  - `/EFI/ubuntu/grubx64.efi`
  - `/EFI/Microsoft/Boot/bootmgfw.efi`

---

### 3. GPU Role
- Still firmware framebuffer (splash logo).
- No acceleration.
- UEFI drawing directly to GPU.

---

### 4. Bootloader Stage Begins
- UEFI loads `grubx64.efi` → into RAM.
- CPU starts executing GRUB.
- Now OS boot stage begins.

---

### 5. GRUB Actions
- Draws splash menu (kernel, recovery, Windows).
- Locates `/boot` partition.
- Loads:
  - Kernel (`vmlinuz`).
  - initramfs.
  - Config (`grub.cfg`).
- Passes control to kernel.

---

### 6. Linux Filesystem Context
- **First time storage touched.**
- `/boot` contains kernel + initramfs.
- `/etc`, `/dev`, `/var` not touched yet.

---

### 7. initramfs Role
- Provides drivers + helpers.
- Needed if root FS is on LVM/RAID/encrypted.

---

### 8. Transition to Kernel
- GRUB builds **kernel cmdline**:
- CPU jumps into kernel entry point.
- Firmware fully out.

---

### 9. GPU State
- Still basic framebuffer (plymouth splash).
- KMS not active yet.

---

## Step 3: Kernel Initialization

### 1. CPU Enters Kernel
- GRUB jumped into kernel entry.
- Kernel uncompresses into RAM.

---

### 2. Early Kernel Setup
- CPU mode changes → long mode (64-bit).
- Paging + interrupts enabled.
- Kernel playground ready.

---

### 3. GPU Role (still basic)
- Framebuffer from firmware.
- Logs visible as text.

---

### 4. initramfs Mounted
- Temporary `/` in RAM.
- Drivers + tools available.

---

### 5. Device Discovery (udev)
- Kernel probes buses.
- `/dev` files created dynamically.

---

### 6. Root Filesystem Switch
- Kernel mounts real `/` from SSD.
- pivot_root → replaces initramfs with real FS.
- Now `/etc`, `/usr`, `/home` available.

---

### 7. Kernel Modules
- Drivers from `/lib/modules/`.
- GPU drivers (e.g., amdgpu, i915).

---

### 8. Kernel Mode Setting (KMS)
- KMS takes GPU control:
- Sets resolution, refresh.
- Allocates framebuffers.
- Screen flicker as resolution changes.

---

### 9. Spawn PID 1
- Kernel executes `/sbin/init` (systemd).
- Kernel done → hands over.

---

## Step 4: init/systemd (PID 1) → Userland Boot

### 1. Kernel Hands Control
- Runs `/sbin/init`.
- This = PID 1.

---

### 2. systemd Startup
- Reads configs `/etc/systemd/system/`.
- Starts services in parallel.
- Uses cgroups for resources.

---

### 3. Decide Target
- Default target:
- `graphical.target` → GUI.
- `multi-user.target` → CLI.
- User can change with `systemctl set-default`.

---

### 4. Start Display Manager (GUI case)
- systemd starts GDM/LightDM.
- Display Manager:
- Starts Xorg or Wayland.
- Draws login screen.
- Authenticates user.

---

### 5. Start Getties (CLI case)
- spawns TTY login prompts.

---

### 6. Filesystem Role
- `/etc/fstab` → mounts.
- `/var/log` → logs.
- `/dev` → devices for daemons.

---

### 7. GPU Role
- Now driven by Wayland/Xorg.
- DRM/KMS still sets resolution.
- GPU VRAM used for textures.

---

## Step 5: User Login → Session Spawn

### 1. User Credentials
- CLI: checked via `/etc/passwd`, `/etc/shadow`.
- GUI: PAM stack authenticates.

---

### 2. Session Start
- `systemd --user` starts.
- User-specific cgroup created.

---

### 3. Environment Setup
- CLI reads: `/etc/profile`, `~/.bashrc`.
- GUI reads: `/etc/X11/xinit`, `/etc/xdg/`, `~/.config/`.

---

### 4. GPU Role
- GUI: gnome-session → Mutter compositor.
- Mutter → Wayland/DRM → GPU renders desktop.
- CLI: text framebuffer only.

---

### 5. Daemons & Services
- dbus-daemon, pulseaudio/pipewire, gvfs, tray apps.

---

### 6. Final Result
- CLI: shell prompt, GPU = text.
- GUI: GNOME desktop, GPU compositing.

---

## Step 6: Desktop Running (Steady State)

### 1. Processes & CPU
- Kernel scheduler balances:
- System services (systemd, journald).
- User services (dbus, sound).
- Applications (browser, editor).

---

### 2. GPU & Compositing
- GPU compositing via Mutter.
- Windows drawn off-screen → textures.
- Blended → output via DRM/KMS → monitor.

---

### 3. Filesystem Roles
- `/home` → user files.
- `/usr/bin` → apps.
- `/etc` → configs.
- `/var/log` → logs.
- `/dev` → devices in use.
- `/proc`, `/sys` → runtime kernel info.

---

### 4. User Interactions
- Input events → `/dev/input/*`.
- Passed kernel → Wayland/Xorg → compositor → app → GPU → monitor.

---

### 5. Background Work
- CPU interrupts.
- Memory flush.
- Cron/systemd-timers.
- Updates.

---

### 6. Security
- PAM → session security.
- SELinux/AppArmor → restrictions.
- cgroups/namespaces → isolation.

---

### 7. What You See
- CLI: stable shell prompt.
- GUI: GNOME desktop fully rendered.

---

# Final Summary
Power switch → firmware (UEFI/BIOS + VBIOS) → bootloader (GRUB) → kernel → init/systemd → login → desktop running.
