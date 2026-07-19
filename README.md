# BarzbugOS — Debian 13 (Trixie) From-Scratch Installation Guide

A complete, command-by-command guide for building a minimal, snapshot-protected
Debian 13 desktop with `debootstrap`: plain Btrfs on two NVMe disks, GRUB +
grub-btrfs bootable snapshots, Snapper wired into every apt/nala transaction, a
full sway desktop, and Steam — while staying strictly inside the
[Don't Break Debian](https://wiki.debian.org/DontBreakDebian) rules.

> **Repo layout:** this README is the full guide.
> [os_architecture.md](os_architecture.md) is the design document (what & why),
> [01_disk_partitioning.md](01_disk_partitioning.md),
> [02_debootstrap_base.md](02_debootstrap_base.md),
> [03_hardening_and_tools.md](03_hardening_and_tools.md) are the condensed
> per-phase checklists.

## Design summary

| Decision | Choice |
|---|---|
| Base | Debian 13 Trixie (stable), `debootstrap --variant=minbase` |
| Init | systemd (PID 1), via `systemd-sysv` |
| Repos | `trixie`, `trixie-updates`, `trixie-security` only — `main contrib non-free non-free-firmware`, deb822 format, i386 multiarch (Steam) |
| Disks | nvme0n1 = ESP + Btrfs system; nvme1n1 = Btrfs /home. **No LUKS, no LVM** |
| /boot | Inside the root subvolume `@` → snapshots contain kernel + initramfs |
| Snapshots | Snapper (root + home) + apt hooks + grub-btrfs; Arch-style rollback (`subvol=@` pinned) |
| Secure Boot | Disabled in firmware |
| Desktop | sway + waybar + wofi + dunst + foot; pipewire; NetworkManager; tty1 autologin |
| Locale | `en_US.UTF-8`, US keyboard, Europe/Istanbul timezone |
| Hardware | Desktop, AMD CPU + AMD GPU (amdgpu/RADV), Ethernet only — no WiFi/BT |

**Accepted trade-offs (deliberate):** no disk encryption + autologin means zero
protection against physical access; zram-only swap means no hibernation.

---

## Table of contents

0. [Preparation](#0-preparation)
1. [Live environment](#1-live-environment)
2. [Partitioning & filesystems](#2-partitioning--filesystems)
3. [Debootstrap](#3-debootstrap)
4. [fstab & chroot entry](#4-fstab--chroot-entry)
5. [Base configuration (inside chroot)](#5-base-configuration-inside-chroot)
6. [Package installation](#6-package-installation)
7. [zram, user, GRUB, autologin](#7-zram-user-grub-autologin)
8. [Leave chroot & first boot](#8-leave-chroot--first-boot)
9. [Snapper + apt hooks + grub-btrfs](#9-snapper--apt-hooks--grub-btrfs)
10. [Security baseline](#10-security-baseline)
11. [Desktop & performance polish](#11-desktop--performance-polish)
12. [Rollback: procedure & mandatory drill](#12-rollback-procedure--mandatory-drill)
13. [Final audit](#13-final-audit)
- [Appendix A: Recovery chroot](#appendix-a-recovery-chroot)
- [Appendix B: What was deliberately left out](#appendix-b-what-was-deliberately-left-out)
- [Appendix C: Troubleshooting — likely failures, stage by stage](#appendix-c-troubleshooting--likely-failures-stage-by-stage)
- [Appendix D: Living with the system — maintenance & upgrades](#appendix-d-living-with-the-system--maintenance--upgrades)

---

## 0. Preparation

1. Download the current **Debian Live Trixie (standard, no DE)** image from
   <https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/>
   (`debian-live-13.*-amd64-standard.iso`).
2. Verify it:
   ```sh
   sha256sum -c --ignore-missing SHA256SUMS
   ```
3. Write to USB (double-check the device!):
   ```sh
   sudo dd if=debian-live-13.*-amd64-standard.iso of=/dev/sdX bs=4M status=progress oflag=sync
   ```
4. Firmware setup: **disable Secure Boot**, boot mode UEFI.
5. Back up anything you still need from both NVMe disks — **they will be wiped**.

## 1. Live environment

Boot the live USB (user `user`, password `live`), become root and get tools:

```sh
sudo -i
apt update
apt install -y debootstrap gdisk dosfstools btrfs-progs
```

Confirm UEFI and identify the disks:

```sh
[ -d /sys/firmware/efi ] && echo UEFI-OK
lsblk -o NAME,SIZE,MODEL,SERIAL
```

Set the disk variables **once** and re-check them — device order can change
between boots. (Do not name the second one `HOME`; that would clobber the
shell's `$HOME`.)

```sh
SYS=/dev/nvme0n1        # system disk → ESP + root btrfs
HOMEDISK=/dev/nvme1n1   # home disk  → home btrfs
```

## 2. Partitioning & filesystems

> ⚠ Point of no return — everything on both disks is destroyed.

```sh
# System disk: 1 GiB ESP + rest Btrfs
sgdisk --zap-all "$SYS"
sgdisk -n1:0:+1GiB -t1:ef00 -c1:EFI    "$SYS"
sgdisk -n2:0:0     -t2:8300 -c2:system "$SYS"

# Home disk: single Btrfs partition
sgdisk --zap-all "$HOMEDISK"
sgdisk -n1:0:0 -t1:8300 -c1:home "$HOMEDISK"

partprobe; udevadm settle

mkfs.vfat  -F32 -n EFI    "${SYS}p1"
mkfs.btrfs -f   -L system "${SYS}p2"
mkfs.btrfs -f   -L home   "${HOMEDISK}p1"
```

Create subvolumes. `/boot` deliberately has **no** subvolume of its own — it
lives inside `@`, so every snapshot carries its matching kernel + initramfs
(this is what makes booting old snapshots actually work). No `@tmp` either:
Trixie mounts `/tmp` as tmpfs by default and we keep that default.

```sh
mount "${SYS}p2" /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
umount /mnt

mount "${HOMEDISK}p1" /mnt
btrfs subvolume create /mnt/@home
umount /mnt
```

Final mount for installation. Options are `noatime,compress=zstd` everywhere;
no `discard` (weekly `fstrim.timer` handles TRIM instead):

```sh
mount -o noatime,compress=zstd,subvol=@          "${SYS}p2"      /mnt
mkdir -p /mnt/{.snapshots,var/log,var/cache,boot/efi,home}
mount -o noatime,compress=zstd,subvol=@snapshots "${SYS}p2"      /mnt/.snapshots
mount -o noatime,compress=zstd,subvol=@var_log   "${SYS}p2"      /mnt/var/log
mount -o noatime,compress=zstd,subvol=@var_cache "${SYS}p2"      /mnt/var/cache
mount -o noatime,compress=zstd,subvol=@home      "${HOMEDISK}p1" /mnt/home
mount -o umask=0077                              "${SYS}p1"      /mnt/boot/efi

findmnt -R /mnt   # verify before continuing
```

## 3. Debootstrap

```sh
debootstrap --arch=amd64 --variant=minbase trixie /mnt http://deb.debian.org/debian
```

`minbase` = Essential + apt and *nothing* else — no init, no network stack, no
locales. Every "obvious" package below is therefore mandatory, not decoration.

Replace the sources with the deb822 file (Trixie's standard). Three suites,
never anything else — `trixie-updates` **is part of stable** (point-release
fixes), this is not FrankenDebian. `non-free` exists solely for
`steam-installer`.

```sh
rm -f /mnt/etc/apt/sources.list

cat > /mnt/etc/apt/sources.list.d/debian.sources <<'EOF'
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
EOF
```

## 4. fstab & chroot entry

Generate fstab **before** entering the chroot (the `$SYS`/`$HOMEDISK` variables
exist only out here):

```sh
UUID_EFI=$( blkid -s UUID -o value "${SYS}p1")
UUID_SYSFS=$(blkid -s UUID -o value "${SYS}p2")
UUID_HOMEFS=$(blkid -s UUID -o value "${HOMEDISK}p1")

cat > /mnt/etc/fstab <<EOF
# <filesystem>     <mountpoint>  <type>  <options>                                  <dump> <pass>
UUID=$UUID_SYSFS   /             btrfs   noatime,compress=zstd,subvol=@             0 0
UUID=$UUID_SYSFS   /.snapshots   btrfs   noatime,compress=zstd,subvol=@snapshots    0 0
UUID=$UUID_SYSFS   /var/log      btrfs   noatime,compress=zstd,subvol=@var_log      0 0
UUID=$UUID_SYSFS   /var/cache    btrfs   noatime,compress=zstd,subvol=@var_cache    0 0
UUID=$UUID_HOMEFS  /home         btrfs   noatime,compress=zstd,subvol=@home         0 0
UUID=$UUID_EFI     /boot/efi     vfat    umask=0077                                 0 1
# /tmp: intentionally absent — Trixie tmpfs default applies
EOF

findmnt --verify --tab-file /mnt/etc/fstab
```

`subvol=@` is pinned on purpose: it is the foundation of the Arch-style
rollback model used in [§12](#12-rollback-procedure--mandatory-drill).

Enter the chroot:

```sh
mount --rbind --make-rslave /dev  /mnt/dev
mount -t proc  proc              /mnt/proc
mount --rbind --make-rslave /sys  /mnt/sys
mount --rbind --make-rslave /run  /mnt/run

# efivarfs must be visible inside the chroot or grub-install/efibootmgr will fail
mountpoint -q /sys/firmware/efi/efivars || \
    mount -t efivarfs efivarfs /sys/firmware/efi/efivars

# The live system's resolv.conf is a symlink into /run — copy the CONTENT
rm -f /mnt/etc/resolv.conf
cp --dereference /etc/resolv.conf /mnt/etc/resolv.conf

chroot /mnt /bin/bash
```

Everything in §5–§7 runs **inside the chroot**.

## 5. Base configuration (inside chroot)

```sh
export DEBIAN_FRONTEND=noninteractive
export LC_ALL=C.UTF-8   # silences "Setting locale failed" warnings until locale-gen runs

# Identity
echo barzbug > /etc/hostname
cat > /etc/hosts <<'EOF'
127.0.0.1   localhost
127.0.1.1   barzbug
::1         localhost ip6-localhost ip6-loopback
EOF

# Multiarch for Steam — must precede the first apt update
dpkg --add-architecture i386
apt update

# Locale: en_US.UTF-8 only (English system)
apt install -y locales
sed -i 's/^# *en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen
update-locale LANG=en_US.UTF-8

# Timezone
ln -sf /usr/share/zoneinfo/Europe/Istanbul /etc/localtime
dpkg-reconfigure -f noninteractive tzdata

# Keyboard: US layout (Debian default — installed with console-setup in §6;
# verify afterwards that /etc/default/keyboard has XKBLAYOUT="us")
```

## 6. Package installation

Keep apt's default `Install-Recommends` **on** — under minbase it quietly pulls
plumbing you'd otherwise chase for days.

```sh
# Boot & kernel (no linux-headers: this build has zero DKMS modules —
# AMD GPU support is the in-kernel amdgpu driver)
apt install -y linux-image-amd64 amd64-microcode grub-efi-amd64 efibootmgr \
    dosfstools btrfs-progs firmware-amd-graphics

# Core system — the minbase gaps.
# systemd-sysv = systemd as PID 1 (the "-sysv" suffix only means it ships the
# /sbin/init compat symlinks; this is the standard Debian systemd boot path).
# libpam-systemd + dbus-user-session are load-bearing: logind sessions (sway
# seat access) and the D-Bus user bus (pipewire, portals, dunst) need them.
apt install -y systemd-sysv libpam-systemd dbus dbus-user-session sudo \
    console-setup keyboard-configuration tzdata systemd-timesyncd \
    systemd-resolved ca-certificates curl wget less man-db bash-completion \
    nano pciutils usbutils smartmontools util-linux-extra

# Network (Ethernet only — no wireless firmware, bluetooth never installed).
# network-manager-gnome is the Debian binary package that contains nm-applet.
apt install -y network-manager network-manager-gnome

# Security & maintenance
apt install -y apparmor apparmor-utils ufw gufw unattended-upgrades \
    needrestart nala

# Snapshots (grub-btrfs itself comes in §9; inotify-tools is its dependency)
apt install -y snapper snapper-gui inotify-tools

# Sway desktop
apt install -y sway swaybg swaylock swayidle waybar wofi dunst foot \
    grim slurp wl-clipboard xwayland \
    xdg-desktop-portal-wlr xdg-desktop-portal-gtk mate-polkit

# Audio
apt install -y pipewire pipewire-pulse wireplumber rtkit pavucontrol

# Fonts & theming
apt install -y fonts-jetbrains-mono fonts-noto-core fonts-noto-color-emoji \
    fonts-font-awesome adwaita-icon-theme

# Desktop apps & QoL tools
apt install -y firefox-esr thunderbird thunar gvfs tumbler imv mpv \
    fastfetch xdg-user-dirs git htop rsync unzip zstd

# Gaming & performance
apt install -y steam-installer mesa-vulkan-drivers mesa-vulkan-drivers:i386 \
    vulkan-tools mesa-utils gamemode zram-tools bpftune power-profiles-daemon \
    profile-sync-daemon
```

## 7. zram, user, GRUB, autologin

```sh
# zram: 50% of RAM, zstd
sed -i -e 's/^#\? *ALGO=.*/ALGO=zstd/' -e 's/^#\? *PERCENT=.*/PERCENT=50/' \
    /etc/default/zramswap

cat > /etc/sysctl.d/99-zram.conf <<'EOF'
# zram-friendly VM tuning
vm.swappiness = 180
vm.page-cluster = 0
EOF

# User (root stays locked — sudo-only administration, Debian convention)
useradd -m -G sudo,video,input -s /bin/bash baris
passwd baris

# GRUB (no cryptodisk, no os-prober — single-OS UEFI machine)
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian
sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=3/' /etc/default/grub
update-grub

# tty1 autologin → sway
mkdir -p /etc/systemd/system/getty@tty1.service.d
cat > /etc/systemd/system/getty@tty1.service.d/autologin.conf <<'EOF'
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin baris --noclear %I $TERM
EOF

cat >> /home/baris/.bash_profile <<'EOF'
# Auto-start sway on tty1 (autologin)
if [ -z "$WAYLAND_DISPLAY" ] && [ "$XDG_VTNR" = "1" ]; then
    export XDG_CURRENT_DESKTOP=sway
    exec sway
fi
EOF
chown baris:baris /home/baris/.bash_profile

# Services for first boot
systemctl enable NetworkManager systemd-timesyncd systemd-resolved \
    fstrim.timer zramswap
```

## 8. Leave chroot & first boot

Pre-flight checks, still inside the chroot:

```sh
apt policy                      # ONLY trixie / trixie-updates / trixie-security
ls /boot/vmlinuz-* /boot/initrd.img-*   # kernel + initramfs exist
efibootmgr -v | grep -i debian  # EFI boot entry present
exit
```

Back in the live shell:

```sh
umount -R /mnt
reboot
```

Remove the USB. The machine should boot straight into sway as `baris`.
If the screen is black-with-cursor: sway is running — open a terminal with
`Super+Enter` (foot). Sections §9–§13 run on the installed system.

## 9. Snapper + apt hooks + grub-btrfs

### 9.1 Snapper configs

**Order matters here.** Our fstab already mounts `@snapshots` at `/.snapshots`,
and `snapper create-config` refuses to run if `/.snapshots` already exists — so
unmount first, create the config, then swap snapper's own subvolume for ours:

```sh
sudo umount /.snapshots
sudo rmdir /.snapshots

sudo snapper -c root create-config /    # creates its own /.snapshots subvolume
sudo btrfs subvolume delete /.snapshots # replace it with our @snapshots:
sudo mkdir /.snapshots
sudo mount /.snapshots                  # fstab entry from §4 takes over
sudo chmod 750 /.snapshots

sudo snapper -c home create-config /home
```

For `home`, keep the nested `/home/.snapshots` subvolume snapper created —
home snapshots then live on the home disk, and nested subvolumes are
automatically excluded from their parent's snapshots. Nothing to fix there.

Tune both configs:

```sh
sudo snapper -c root set-config ALLOW_USERS=baris TIMELINE_CREATE=yes \
    TIMELINE_LIMIT_HOURLY=5 TIMELINE_LIMIT_DAILY=7 TIMELINE_LIMIT_WEEKLY=2 \
    TIMELINE_LIMIT_MONTHLY=0 TIMELINE_LIMIT_YEARLY=0 NUMBER_LIMIT=20
sudo snapper -c home set-config ALLOW_USERS=baris TIMELINE_CREATE=yes \
    TIMELINE_LIMIT_HOURLY=3 TIMELINE_LIMIT_DAILY=7 TIMELINE_LIMIT_WEEKLY=1 \
    TIMELINE_LIMIT_MONTHLY=0 TIMELINE_LIMIT_YEARLY=0

sudo systemctl enable --now snapper-timeline.timer snapper-cleanup.timer
```

### 9.2 Apt/nala transaction snapshots

Debian's snapper ships **no** apt hook — we add our own `DPkg::Pre-Invoke` /
`Post-Invoke` pair. It fires for apt, nala *and* unattended-upgrades (they all
drive libapt/dpkg). apt may invoke dpkg several times per transaction; the
`/run` marker collapses that into one pre/post snapshot pair per apt run.

```sh
sudo tee /usr/local/sbin/snapper-apt-pre.sh >/dev/null <<'EOF'
#!/bin/sh
# One pre-snapshot per apt transaction (marker guards repeat dpkg invocations)
[ -f /run/snapper-apt-pre-num ] && exit 0
snapper -c root create -t pre -c number -p -d "apt transaction" \
    > /run/snapper-apt-pre-num 2>/dev/null || rm -f /run/snapper-apt-pre-num
exit 0
EOF

sudo tee /usr/local/sbin/snapper-apt-post.sh >/dev/null <<'EOF'
#!/bin/sh
[ -f /run/snapper-apt-pre-num ] || exit 0
N=$(cat /run/snapper-apt-pre-num)
rm -f /run/snapper-apt-pre-num
[ -n "$N" ] && snapper -c root create -t post --pre-number "$N" -c number \
    -d "apt transaction" 2>/dev/null
exit 0
EOF

sudo chmod +x /usr/local/sbin/snapper-apt-pre.sh /usr/local/sbin/snapper-apt-post.sh

sudo tee /etc/apt/apt.conf.d/80snapper >/dev/null <<'EOF'
DPkg::Pre-Invoke  { "/usr/local/sbin/snapper-apt-pre.sh  || true"; };
DPkg::Post-Invoke { "/usr/local/sbin/snapper-apt-post.sh || true"; };
EOF
```

Test: `sudo nala install sl` → `sudo snapper -c root list` must show a
pre/post pair.

### 9.3 grub-btrfs (the only out-of-repo software on this system)

Not packaged in Debian — installed from source, from a **release tag**, with
the installed file list recorded for clean removal:

```sh
sudo apt install -y make gawk
git clone https://github.com/Antynea/grub-btrfs.git
cd grub-btrfs
git checkout "$(git describe --tags --abbrev=0)"   # latest release tag, not master
sudo make install 2>&1 | tee ~/grub-btrfs-installed-files.txt
```

Keep `grub-btrfs-installed-files.txt` (commit it to this repo). It installs
into `/etc/grub.d/41_snapshots-btrfs`, `/etc/default/grub-btrfs/`,
`/usr/bin/grub-btrfsd` and a systemd unit.

```sh
sudo systemctl enable --now grub-btrfsd   # watches /.snapshots by default
sudo update-grub                           # "Debian snapshots" submenu appears
```

## 10. Security baseline

```sh
# Firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw enable
sudo systemctl enable ufw

# AppArmor — Debian's default LSM; verify profiles are enforced
sudo aa-status

# Unattended security upgrades
sudo tee /etc/apt/apt.conf.d/20auto-upgrades >/dev/null <<'EOF'
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
EOF
sudo unattended-upgrade --dry-run --debug   # verify: trixie-security origin active
```

Every unattended security upgrade also gets a snapper pre/post pair via the
§9.2 hooks — automatic updates are rollback-able too.

## 11. Desktop & performance polish

### 11.1 sway session basics

```sh
mkdir -p ~/.config/sway && cp /etc/sway/config ~/.config/sway/config
xdg-user-dirs-update
```

Add to `~/.config/sway/config`:

```
set $term foot
input type:keyboard xkb_layout us

# CRITICAL for portals/screen-sharing/pipewire: hand the session environment
# to systemd user services & D-Bus activation. Without this, screen sharing
# and xdg-desktop-portal-wlr silently fail.
exec dbus-update-activation-environment --systemd \
    WAYLAND_DISPLAY XDG_CURRENT_DESKTOP=sway

# System tray & desktop plumbing
exec nm-applet
exec /usr/libexec/mate-polkit/polkit-mate-authentication-agent-1
# ^ if the path differs: dpkg -L mate-polkit | grep agent

# Idle & lock: lock after 10 min, screen off after 15
exec swayidle -w \
    timeout 600 'swaylock -f -c 000000' \
    timeout 900 'swaymsg "output * power off"' \
        resume  'swaymsg "output * power on"' \
    before-sleep 'swaylock -f -c 000000'

# Volume & screenshot keys
bindsym XF86AudioRaiseVolume exec wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%+
bindsym XF86AudioLowerVolume exec wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-
bindsym XF86AudioMute        exec wpctl set-mute   @DEFAULT_AUDIO_SINK@ toggle
bindsym Print exec grim -g "$(slurp)" ~/Pictures/shot-$(date +%s).png

bar swaybar_command waybar
```

Waybar: copy the defaults
(`cp -r /etc/xdg/waybar ~/.config/waybar`) and make sure the `"tray"` module is
present in `modules-right`, otherwise nm-applet's icon has nowhere to appear.

Verify after re-login (`Super+Shift+c` reloads): audio
(`wpctl status` — pipewire), screenshots (`grim -g "$(slurp)"`), screen-share
portal (test in Firefox), tray shows nm-applet.

### 11.2 systemd-oomd (works off the zram swap signal)

```sh
sudo mkdir -p /etc/systemd/system/-.slice.d /etc/systemd/system/user@.service.d

sudo tee /etc/systemd/system/-.slice.d/10-oomd.conf >/dev/null <<'EOF'
[Slice]
ManagedOOMSwap=kill
EOF

sudo tee /etc/systemd/system/user@.service.d/10-oomd.conf >/dev/null <<'EOF'
[Service]
ManagedOOMMemoryPressure=kill
ManagedOOMMemoryPressureLimit=50%
EOF

sudo systemctl enable --now systemd-oomd
sudo systemctl daemon-reload
```

### 11.3 Remaining services & tools

```sh
# BPF auto-tuning + power profiles (both from Debian repos)
sudo systemctl enable --now bpftune power-profiles-daemon

# DNS via systemd-resolved (enabled in §7) — point NetworkManager at it
sudo tee /etc/NetworkManager/conf.d/dns.conf >/dev/null <<'EOF'
[main]
dns=systemd-resolved
EOF
sudo systemctl restart NetworkManager
resolvectl status

# profile-sync-daemon (browser profile in RAM) — per user
systemctl --user enable --now psd
# Optional overlayfs mode: set USE_OVERLAYFS="yes" in ~/.config/psd/psd.conf and
# add the sudoers line psd tells you about (psd preview shows it), then:
# systemctl --user restart psd

# TRIM sanity check (fstrim.timer already enabled in §7)
sudo fstrim -av
lsblk --discard

# Boot-time review — investigate outliers, do NOT bulk-disable units
systemd-analyze blame
systemd-analyze critical-chain
```

### 11.4 Gaming verification

```sh
vulkaninfo --summary   # RADV / AMD GPU listed
glxinfo -B             # renderer = AMD
gamemoded -t           # gamemode self-test passes
steam                  # first run bootstraps the client (steam-installer)
```

## 12. Rollback: procedure & mandatory drill

This build uses the **Arch-style** model: fstab pins `subvol=@`, grub-btrfs
boots snapshots read-only for inspection, and restoring = swapping a snapshot
into `@`. (`snapper rollback` relies on the default-subvolume mechanism and is
**not** used here.)

### Procedure

1. Reboot → GRUB → **Debian snapshots** submenu → boot the snapshot you trust
   (read-only) and verify it is good.
2. From that boot (or the live USB), mount the Btrfs top-level:
   ```sh
   sudo mount -o subvolid=5 /dev/nvme0n1p2 /mnt
   ```
3. Swap the broken root out, clone the good snapshot in (as read-write):
   ```sh
   sudo mv /mnt/@ /mnt/@broken
   sudo btrfs subvolume snapshot /mnt/.snapshots/<N>/snapshot /mnt/@
   ```
4. `sudo reboot` into the restored system, then regenerate GRUB's menu
   (`sudo update-grub`) and delete `@broken` once satisfied:
   ```sh
   sudo mount -o subvolid=5 /dev/nvme0n1p2 /mnt
   sudo btrfs subvolume delete /mnt/@broken
   ```

### Drill (do this once, on the fresh install, BEFORE trusting the setup)

```sh
sudo snapper -c root create -d "pre-drill"
sudo mv /etc/hostname /etc/hostname.bak        # the "breakage"
# → follow the procedure above to restore the pre-drill snapshot
cat /etc/hostname                               # must say: barzbug
```

Keep the live USB. A rollback system you have never rolled back is a theory,
not a safety net.

## 13. Final audit

```sh
# 1. Repo purity — every line must say trixie/trixie-updates/trixie-security, prio 500
apt policy

# 2. Manual package review — everything here should be recognizable from this guide
apt-mark showmanual | sort

# 3. Only out-of-repo software: grub-btrfs (file list committed to this repo)
cat ~/grub-btrfs-installed-files.txt

# 4. Snapshot lifecycle
sudo snapper -c root list        # timeline + apt pre/post pairs accumulating
systemctl list-timers | grep -E 'snapper|fstrim|apt'

# 5. System health
systemctl --failed               # must be empty
sudo aa-status                   # AppArmor enforcing
sudo ufw status verbose          # active, deny incoming
```

Cold-boot test: power off, power on → lands in sway with network, audio,
portals and tray working, no manual intervention.

---

## Appendix A: Recovery chroot

If the system won't boot and even snapshots don't help, from the live USB:

```sh
sudo -i
mount -o noatime,compress=zstd,subvol=@ /dev/nvme0n1p2 /mnt
mount -o noatime,compress=zstd,subvol=@var_log   /dev/nvme0n1p2 /mnt/var/log
mount -o noatime,compress=zstd,subvol=@var_cache /dev/nvme0n1p2 /mnt/var/cache
mount /dev/nvme0n1p1 /mnt/boot/efi
mount --rbind --make-rslave /dev /mnt/dev
mount -t proc proc /mnt/proc
mount --rbind --make-rslave /sys /mnt/sys
mount --rbind --make-rslave /run /mnt/run
cp /etc/resolv.conf /mnt/etc/resolv.conf
chroot /mnt /bin/bash
# typical repairs: update-grub, grub-install, apt reinstall linux-image-amd64
```

## Appendix B: What was deliberately left out

| Item | Why it's not here |
|---|---|
| LUKS2 / disk encryption | Design decision (home desktop). Historical note: the original plan (Argon2id + GRUB-unlocked /boot) could never have booted — Trixie's GRUB 2.12 cannot open Argon2id LUKS2; that landed in GRUB 2.14. |
| Secure Boot / sbctl | Disabled in firmware. sbctl isn't in Debian repos and custom keys demand re-signing hooks on every kernel/GRUB update — an unbootable-system trap. |
| Separate /boot partition | Would exclude kernels from snapshots → booting old snapshots breaks on kernel/module mismatch. |
| `@tmp` subvolume | Trixie's /tmp is tmpfs by default; keeping Debian defaults is the point. |
| `linux-headers-amd64` | Zero DKMS modules in this build (amdgpu is in-kernel). Install only when a DKMS need appears. |
| `checkinstall` | Abandoned since 2017, pollutes dpkg state — the opposite of Don't Break Debian. |
| low_latency_layer, dmemcg-booster (GitHub tweak scripts) | Unaudited third-party scripts touching sysctl/cgroups. gamemode + zram + bpftune (all from Debian repos) cover the goal. |
| cups, bluetooth, wifi firmware | Hardware/needs don't exist here (Ethernet-only desktop, no printer). Not installed rather than installed-then-disabled. |
| Third-party apt repos, backports, testing/unstable pins | FrankenDebian. If something is missing: Flatpak for apps, or wait for the next stable. |

## Appendix C: Troubleshooting — likely failures, stage by stage

Format: **Symptom** → cause → fix. Search this section before searching the web.

### C.1 Live environment & partitioning

**USB won't boot / boots the old OS** → firmware boot order, or CSM/legacy mode
grabbed it → open the firmware boot menu (usually F8/F11/F12), pick the
`UEFI:`-prefixed USB entry; disable CSM. If the stick itself is suspect, re-`dd`
it and try another port.

**`[ -d /sys/firmware/efi ]` prints nothing (legacy boot)** → the live session
booted in BIOS mode → reboot, choose the UEFI entry. Do NOT continue; the GRUB
install in §7 is UEFI-only.

**`${SYS}p1: No such file or directory` right after sgdisk** → the kernel
hasn't re-read the partition table yet → `partprobe; udevadm settle; lsblk`.
If it persists: `blockdev --rereadpt "$SYS"` or reboot the live session.

**`Device or resource busy` from sgdisk/mkfs** → something on the disk is in
use → `swapoff -a; umount -R /mnt 2>/dev/null`; stale RAID/LVM/filesystem
signatures: `wipefs -a "$SYS"` (same for `$HOMEDISK`), then repeat §2.

**Disk names swapped after a reboot (nvme0n1 ↔ nvme1n1)** → NVMe enumeration
is not stable → re-check `lsblk -o NAME,SIZE,MODEL,SERIAL` and reset `$SYS` /
`$HOMEDISK` before every session. The installed system is immune (fstab uses
UUIDs).

**No network in the live session** → `ip a` shows the interface down →
`dhclient <iface>` or `nmcli device connect <iface>`. Ethernet on this desktop
needs no firmware, so a dead link is cable/switch/DHCP, not drivers.

### C.2 debootstrap & chroot

**debootstrap dies mid-download** → network hiccup; a half-populated target is
not safely resumable → clean restart:
```sh
umount -R /mnt
mount "${SYS}p2" /mnt
btrfs subvolume delete /mnt/@ ; btrfs subvolume create /mnt/@
umount /mnt
# re-do the final mount block of §2, then rerun debootstrap
```

**GPG / "release signed by unknown key" errors** → live image's keyring is
older than the archive → `apt update && apt install -y debian-archive-keyring`
in the live session, retry.

**Inside chroot: `perl: warning: Setting locale failed`** → normal until
`locale-gen` has run; the `export LC_ALL=C.UTF-8` at the top of §5 silences it.
Harmless either way.

**Inside chroot: DNS broken (`Temporary failure resolving deb.debian.org`)** →
`/etc/resolv.conf` was copied as a dangling symlink → exit chroot, re-do the
`rm -f` + `cp --dereference` lines from §4, re-enter.

**tzdata/console-setup ask interactive questions anyway** →
`DEBIAN_FRONTEND=noninteractive` wasn't exported in *this* shell (it does not
survive exiting/re-entering chroot) → re-export, or just answer the prompts.

### C.3 GRUB & first reboot

**`grub-install: error: efivarfs not found / EFI variables not supported`** →
efivarfs isn't mounted inside the chroot → run the `mountpoint -q ... ||
mount -t efivarfs ...` line from §4 (from outside, or inside the chroot).
Last resort: `grub-install --target=x86_64-efi --efi-directory=/boot/efi
--removable --no-nvram` — installs to the fallback path
`EFI/BOOT/BOOTX64.EFI`, which every firmware boots without an NVRAM entry.

**Reboot lands in the firmware / "no bootable device"** → NVRAM entry missing
or boot order wrong → check `efibootmgr -v`; fix order with
`efibootmgr -o XXXX,...`; or use the `--removable` fallback above.

**`grub rescue>` prompt** → GRUB can't find its modules/root → boot the live
USB, do the [recovery chroot](#appendix-a-recovery-chroot), re-run
`grub-install` + `update-grub`. Usual cause: ESP wasn't mounted at `/boot/efi`
during §7.

**`update-grub` reports no kernels** → `/boot` is empty →
`apt reinstall linux-image-amd64` inside the chroot, then `update-grub`.

**Black screen immediately after GRUB (no console ever appears)** → amdgpu
init without firmware → at the GRUB menu press `e`, append `nomodeset` to the
`linux` line, boot to console, then `apt reinstall firmware-amd-graphics &&
update-initramfs -u && reboot`.

**Emergency mode: "Dependency failed for /home" (or /.snapshots …)** → fstab
typo or wrong UUID → in the emergency shell: `journalctl -xb | grep -i mount`
to see which line, fix `/etc/fstab` (nano works there), `systemctl
daemon-reload`, `exit` or reboot. Cross-check with `blkid`.

### C.4 sway session

**Autologin works but screen stays on a bare console (sway never starts)** →
`.bash_profile` block not reached → check the file is owned by baris and the
`XDG_VTNR` test matches (`echo $XDG_VTNR` on tty1 must print `1`). Run `sway`
manually to see the real error.

**sway exits instantly / `Cannot create XDG_RUNTIME_DIR`** → no logind session
→ `libpam-systemd` and `dbus-user-session` must be installed (§6) and you must
log in on the tty (not via `su`). Verify with `loginctl` — your user must have
a session with `seat0`.

**Config edits break sway** → validate before reload:
`sway -C ~/.config/sway/config` prints the offending line.

**No sound** → `systemctl --user status pipewire wireplumber pipewire-pulse`;
if "Failed to connect to bus": `dbus-user-session` missing or you're testing
via sudo/ssh instead of the desktop session. `wpctl status` must list the sink.

**Screen sharing shows no windows / portal errors** → the environment handoff
is missing → confirm the `dbus-update-activation-environment` exec line from
§11.1 exists, re-login, then check
`systemctl --user status xdg-desktop-portal xdg-desktop-portal-wlr`.

**No tray icons (nm-applet invisible)** → waybar config lacks the `"tray"`
module → add it to `modules-right`, `killall waybar` (sway restarts it).

**gufw / snapper-gui never show their admin prompt** → no polkit agent in the
session → the mate-polkit exec line from §11.1; verify the binary path with
`dpkg -L mate-polkit | grep agent`.

### C.5 Snapper & grub-btrfs

**`create-config` fails: "'.snapshots' already exists"** → you ran it while
`@snapshots` was still mounted → follow §9.1 exactly (umount → rmdir →
create-config → swap subvolumes).

**No pre/post pairs after apt runs** → hook scripts not firing →
`apt-config dump | grep -i invoke` must show both scripts; check they are
executable; run `/usr/local/sbin/snapper-apt-pre.sh` by hand and read the
error. A stale `/run/snapper-apt-pre-num` (from a crashed run) suppresses new
pre-snapshots — delete it.

**GRUB has no "snapshots" submenu** → there were zero snapshots at
`update-grub` time, or grub-btrfsd isn't watching → create one
(`sudo snapper -c root create -d test`), check
`systemctl status grub-btrfsd`, run `sudo update-grub` again.

**Booted a snapshot and "everything is broken/read-only"** → expected:
snapshot boots are read-only inspection environments; services that need to
write will fail. Look around, confirm it's the state you want, then do the
[§12 restore](#12-rollback-procedure--mandatory-drill) — don't try to "use"
a snapshot boot.

**After a restore the GRUB menu looks stale / lists wrong kernels** → the menu
is generated from the *current* `@`'s `/boot` → `sudo update-grub` after every
restore (it's step 4 of the procedure for exactly this reason).

**`btrfs subvolume delete /mnt/@broken` refuses ("directory not empty" is NOT
an error btrfs gives — but "parent subvolume" complaints are)** → you created
a nested subvolume inside `@` at some point → `btrfs subvolume list /mnt` and
delete children first. In this layout `@` has no nested subvolumes by design.

### C.6 Steam & gaming

**`vulkaninfo` shows only `llvmpipe`** → GPU firmware/driver not active →
`dmesg | grep -i amdgpu` (look for firmware load errors),
`apt reinstall firmware-amd-graphics`, reboot.

**Steam launches but games have no 32-bit GL/Vulkan** →
`apt install libgl1:i386 libvulkan1:i386 mesa-vulkan-drivers:i386` (the last
one is already in §6; the first two normally arrive as steam-installer
dependencies).

**`gamemoded -t` fails with D-Bus errors** → you ran it via sudo or outside
the desktop session → run as your normal user inside sway.

**Downloads crawl / disk usage looks weird for game libraries** → CoW +
compression on huge game files is fine in practice, but if you want a
no-snapshot game library, create a dedicated subvolume so it stays out of
root/home snapshots:
`btrfs subvolume create /home/baris/Games` (nested subvolume = excluded from
`@home` snapshots automatically).

## Appendix D: Living with the system — maintenance & upgrades

### D.1 Routine

```sh
sudo nala upgrade          # interactive updates (snapshots happen automatically)
sudo nala autoremove       # after big removals
systemctl --failed         # occasionally; must stay empty
```

Security updates install themselves (unattended-upgrades) and are snapshot-
bracketed by the §9.2 hooks. `needrestart` tells you when a reboot is due
(kernel updates).

### D.2 Adding software — the decision tree (Don't Break Debian, forever)

1. **Debian repo first**: `nala search <thing>`. If it's there, done.
2. **Flatpak second** (GUI apps not in trixie or too old):
   ```sh
   sudo apt install flatpak
   flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo
   ```
   Portals for sway are already installed (§6), so Flatpak apps integrate fine.
3. **/usr/local third** (CLI tools, source builds): keep every out-of-repo file
   under `/usr/local` and record what you put there (grub-btrfs is the
   template: file list committed to this repo).
4. **Never**: third-party apt repos/PPAs, `sudo pip/npm install` into `/usr`,
   `curl | sudo bash`, packages from other Debian releases. That's how
   FrankenDebian starts.

### D.3 Btrfs health (monthly-ish)

```sh
sudo btrfs scrub start /       # then: sudo btrfs scrub status /
sudo btrfs scrub start /home
sudo btrfs device stats /      # error counters — all zeros or investigate
sudo btrfs filesystem usage /  # trust this, not df, on btrfs
```

Prefer automation from the repo: `sudo apt install btrfsmaintenance` and set
`BTRFS_SCRUB_PERIOD=monthly` in `/etc/default/btrfsmaintenance`. Balance is
rarely needed; don't run it recreationally.

### D.4 Snapshot space management

If disk usage creeps up: `sudo snapper -c root list` — old number-cleanup
snapshots are the usual suspects. Lower `NUMBER_LIMIT` / `TIMELINE_LIMIT_*`
(§9.1) and force a pass:
`sudo systemctl start snapper-cleanup.service`. Single-file rescues without a
full rollback: `sudo snapper -c root status <pre>..<post>` then
`sudo snapper -c root undochange <pre>..<post> <path>`.

### D.5 grub-btrfs updates

Occasionally (a few times a year):
```sh
cd ~/grub-btrfs && git fetch --tags
git checkout "$(git describe --tags --abbrev=0)"
sudo make install 2>&1 | tee ~/grub-btrfs-installed-files.txt
sudo systemctl daemon-reload && sudo systemctl restart grub-btrfsd
```

### D.6 Point releases and the Debian 14 upgrade

**Point releases (13.1, 13.2, …):** nothing to do — they arrive through
`trixie` + `trixie-updates` automatically.

**Debian 14 "Forky" (when it is RELEASED as stable, not before):**
1. Read the official release notes first (www.debian.org/releases/forky).
2. Get current: `sudo nala upgrade`, reboot if needrestart says so.
3. Safety net: `sudo snapper -c root create -d "pre-forky"` + verify you can
   still boot snapshots from GRUB.
4. Switch suites:
   ```sh
   sudo sed -i 's/trixie/forky/g' /etc/apt/sources.list.d/debian.sources
   ```
5. Upgrade in two stages:
   ```sh
   sudo apt update
   sudo apt upgrade --without-new-pkgs
   sudo apt full-upgrade
   sudo reboot
   sudo apt autoremove --purge
   ```
6. Re-verify the out-of-repo piece: grub-btrfsd running, snapshot submenu
   regenerates (`sudo update-grub`); if it broke, reinstall per D.5.
7. Keep the snapshot until you've lived on Forky for a week or two.

### D.7 When something feels wrong

```sh
systemctl --failed
journalctl -p err -b        # this boot's errors only
sudo snapper -c root list   # what changed recently? diff it:
sudo snapper -c root status <N>..0
```

And the standing safety rules: the live USB stays in a drawer, this repo stays
cloned somewhere off-machine, and **snapshots are not backups** — anything
irreplaceable in `/home` needs a real copy elsewhere (btrfs send/receive to an
external disk, or plain rsync).
