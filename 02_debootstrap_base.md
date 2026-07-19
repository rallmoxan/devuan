# Step 2: Base System Installation (Debootstrap)

## Objective
Deploy Debian 13 Trixie onto /mnt (prepared in step 1), configure it in chroot,
and make it boot on its own: GRUB, network, user, sway autologin.

## Instructions for Claude Code

### 1. Bootstrap
```
debootstrap --arch=amd64 --variant=minbase trixie /mnt http://deb.debian.org/debian
```
Reminder: minbase = Essential + apt ONLY. No init, no network stack, no locales —
everything below is mandatory, not optional.

### 2. APT sources (deb822 — Trixie standard)
Remove any `/mnt/etc/apt/sources.list` debootstrap left behind. Create
`/mnt/etc/apt/sources.list.d/debian.sources`:
```
Types: deb
URIs: http://deb.debian.org/debian
Suites: trixie trixie-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: http://security.debian.org/debian-security
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```
Policy check: these three suites and nothing else, ever (No FrankenDebian).
`non-free` is here for steam-installer only.

### 3. Chroot
Bind /dev, /dev/pts, /proc, /sys, /run (and efivarfs if needed), copy
/etc/resolv.conf, then chroot. All following steps run inside.

### 4. Identity & basics
- `dpkg --add-architecture i386` (BEFORE first apt update)
- `apt update`
- hostname: `barzbug` (+ matching `/etc/hosts`: `127.0.1.1 barzbug`)
- locales: `en_US.UTF-8` ONLY (English system), set as default
- timezone: `Europe/Istanbul`
- keyboard: **US layout** (`XKBLAYOUT="us"` — Debian default, verify in
  `/etc/default/keyboard`)

### 5. fstab
Build from `blkid` UUIDs, mirroring the step-1 table exactly
(`subvol=@...` pinned — required by the Arch-style rollback model).
ESP line: `umask=0077`. `/tmp`: NO entry (Trixie tmpfs default).

### 6. Packages (grouped; single nala/apt transaction per group is fine)

**Boot & kernel:**
`linux-image-amd64 amd64-microcode grub-efi-amd64 efibootmgr dosfstools btrfs-progs firmware-amd-graphics`
(No linux-headers: no DKMS in this build.)

**Core system (minbase gaps):**
`systemd-sysv libpam-systemd dbus dbus-user-session sudo locales console-setup keyboard-configuration tzdata systemd-timesyncd systemd-resolved ca-certificates curl wget less man-db bash-completion nano pciutils usbutils smartmontools util-linux-extra`
- `systemd-sysv` = **systemd as PID 1** (the "-sysv" suffix only means it ships the
  `/sbin/init` compat symlinks — this is how every standard Debian systemd install
  boots; no sysvinit anywhere).
- `libpam-systemd` + `dbus-user-session` are load-bearing under minbase: logind user
  sessions (sway seat access) and the D-Bus user bus (pipewire, portals, dunst) need
  them.

**Network (Ethernet only — no wireless firmware, no bluetooth):**
`network-manager network-manager-gnome`
(`network-manager-gnome` = the nm-applet binary package in Debian.)

**Security & maintenance:**
`apparmor apparmor-utils ufw gufw unattended-upgrades needrestart nala`

**Snapshots:**
`snapper snapper-gui inotify-tools`
(inotify-tools = grub-btrfsd dependency; grub-btrfs itself is installed in step 3.)

**Sway desktop:**
`sway swaybg swaylock swayidle waybar wofi dunst foot grim slurp wl-clipboard xwayland xdg-desktop-portal-wlr xdg-desktop-portal-gtk mate-polkit`

**Audio:**
`pipewire pipewire-pulse wireplumber rtkit pavucontrol`

**Fonts & theming:**
`fonts-jetbrains-mono fonts-noto-core fonts-noto-color-emoji fonts-font-awesome adwaita-icon-theme`

**Desktop apps:**
`firefox-esr thunderbird thunar gvfs tumbler imv mpv fastfetch`

**Gaming & performance:**
`steam-installer mesa-vulkan-drivers mesa-vulkan-drivers:i386 vulkan-tools mesa-utils gamemode zram-tools bpftune power-profiles-daemon profile-sync-daemon`

### 7. zram
`/etc/default/zramswap`: `ALGO=zstd`, `PERCENT=50`.
`/etc/sysctl.d/99-zram.conf`:
```
vm.swappiness = 180
vm.page-cluster = 0
```

### 8. User
- `useradd -m -G sudo,video,input -s /bin/bash baris` + password.
- Root account: locked (sudo-only administration, Debian convention).

### 9. GRUB
- `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian`
- `update-grub`
- No cryptodisk, no os-prober needed (single-OS machine).

### 10. Autologin → sway (tty1)
- `/etc/systemd/system/getty@tty1.service.d/autologin.conf`:
  agetty `--autologin baris --noclear`.
- In `~/.bash_profile`: if on tty1 and no wayland session → `exec sway`.
- ⚠ Documented trade-off: no encryption + autologin = anyone at the machine is in.

### 11. Enable services (systemctl enable, effective on first boot)
`NetworkManager systemd-timesyncd fstrim.timer zramswap` (ufw + the rest in step 3).

### 12. Exit checklist before reboot
- `apt policy` inside chroot: only trixie / trixie-updates / trixie-security.
- fstab UUIDs verified against blkid; `findmnt --verify` clean.
- EFI boot entry exists (`efibootmgr -v` shows "debian").
- initramfs exists for the installed kernel (`ls /boot/initrd.img-*`).
