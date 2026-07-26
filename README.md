# BarzbugOS — Debian 13 (Trixie) From-Scratch Installation Guide

A complete, command-by-command guide for building a minimal, snapshot-protected
Debian 13 desktop with `debootstrap`: plain Btrfs on two NVMe disks, GRUB +
grub-btrfs bootable snapshots, Snapper wired into every apt/nala transaction, a
**minimal GNOME desktop (`gnome-core`)**, and **Flatpak/Flathub** as the single
escape hatch for everything the archive doesn't ship — while staying strictly
inside the [Don't Break Debian](https://wiki.debian.org/DontBreakDebian) rules.

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
| Repos | `trixie`, `trixie-updates`, `trixie-security` only — all four components (`main contrib non-free non-free-firmware`), deb822 format, **https throughout**, mirror chosen by `nala fetch`. **No i386 multiarch** |
| Disks | nvme0n1 = ESP + Btrfs system; nvme1n1 = Btrfs /home. **No LUKS, no LVM** |
| /boot | Inside the root subvolume `@` → snapshots contain kernel + initramfs |
| Snapshots | Snapper (root + home) + apt hooks + grub-btrfs; Arch-style rollback (`subvol=@` pinned) |
| Secure Boot | Disabled in firmware |
| Desktop | `gnome-core` (GNOME 48) on Wayland, GDM3 login **with password**; PipeWire; NetworkManager |
| Apps | Debian repo first, **Flathub (system-wide) second**. Steam = `com.valvesoftware.Steam` (Flatpak) |
| Locale | `en_US.UTF-8`, US keyboard, Europe/Istanbul timezone |
| Hardware | Desktop, AMD CPU + AMD GPU (amdgpu/RADV), Ethernet only — no WiFi/BT |

**Accepted trade-offs (deliberate):** no disk encryption means zero protection
against physical access (the GDM password only protects the running session);
zram-only swap means no hibernation; Flathub is a third-party *application*
source outside dpkg — sandboxed, never mixed into apt, and deliberately the
*only* such source.

---

## Table of contents

0. [Preparation](#0-preparation)
1. [Live environment](#1-live-environment)
2. [Partitioning & filesystems](#2-partitioning--filesystems)
3. [Debootstrap](#3-debootstrap)
4. [fstab & chroot entry](#4-fstab--chroot-entry)
5. [Base configuration (inside chroot)](#5-base-configuration-inside-chroot)
6. [Package installation](#6-package-installation)
7. [zram, user, GRUB, graphical target](#7-zram-user-grub-graphical-target)
8. [Leave chroot & first boot](#8-leave-chroot--first-boot)
9. [Snapper + apt hooks + grub-btrfs](#9-snapper--apt-hooks--grub-btrfs)
10. [Security baseline](#10-security-baseline)
11. [GNOME, Flatpak, Steam & performance polish](#11-gnome-flatpak-steam--performance-polish)
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

### 1.1 Pick the fastest mirror (nala fetch) — do this before debootstrap

Everything downloaded from here on (the bootstrap itself and the ~2–3 GB of
packages in §6) comes from whatever mirror we choose now, so this is the single
highest-leverage speed decision in the install. `nala fetch` benchmarks the
official mirror list and ranks them:

```sh
apt install -y nala ca-certificates
nala fetch --debian trixie --https-only --non-free --auto --fetches 3 \
    -c TR -c DE -c NL
```

- `--https-only` — only mirrors that serve TLS, which is what we want for the
  whole install (see below).
- `--auto` — no interactive picker; `--fetches 3` keeps the three fastest.
- `-c` filters by country (ISO code, repeatable). Drop the `-c` flags entirely
  if the filter comes back with too few https mirrors — nala measures latency
  anyway, and a fast German mirror beats a slow local one.
- If any flag is rejected (this is nala 0.16.0 in Trixie; older builds differ),
  plain `nala fetch --debian trixie --https-only` opens the interactive picker
  and gets you to the same place one keypress later.

`nala fetch` writes `/etc/apt/sources.list.d/nala-sources.list` **in the live
session** (which is throwaway). We only use it to *discover* the mirror; the
installed system gets a single hand-written deb822 file in §3, so it never ends
up with two overlapping sources files:

```sh
MIRROR=$(awk '/^deb /{print $2; exit}' /etc/apt/sources.list.d/nala-sources.list)
MIRROR=${MIRROR:-https://deb.debian.org/debian}    # fallback
echo "$MIRROR"    # must print an https:// URL — keep this shell open until §3
```

> **Mirror trade-off, stated honestly:** `deb.debian.org` is not a server, it's a
> redirector to a CDN — usually already fast and *always* in sync. A hand-picked
> mirror can beat it on throughput but can also lag behind during point releases
> (symptom: "Release file is not valid yet" or hash-sum mismatches — see
> [C.2](#c2-debootstrap--chroot)). If that ever happens, switching the URI back
> to `https://deb.debian.org/debian` is a one-line fix.

> **Why https:** apt verifies everything with OpenPGP signatures, so http is not
> *insecure* — Debian's own default is http for a reason (proxy/cache
> friendliness). What TLS adds here is transport privacy: which packages you
> install stops being visible to anyone between you and the mirror. That is the
> reason to do it, and it costs nothing measurable.

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

`@var_lib_flatpak` is new in this revision: Flatpak runtimes and apps are
multi-gigabyte and update on their own schedule. Keeping them in their own
subvolume means root snapshots stay small and a system rollback does not drag
your applications back in time with it (Flatpak has its own history — see
[§12](#12-rollback-procedure--mandatory-drill) and [D.5](#d5-flatpak-upkeep)).

```sh
mount "${SYS}p2" /mnt
btrfs subvolume create /mnt/@
btrfs subvolume create /mnt/@snapshots
btrfs subvolume create /mnt/@var_log
btrfs subvolume create /mnt/@var_cache
btrfs subvolume create /mnt/@var_lib_flatpak
umount /mnt

mount "${HOMEDISK}p1" /mnt
btrfs subvolume create /mnt/@home
umount /mnt
```

Final mount for installation. Options are `noatime,compress=zstd` everywhere;
no `discard` (weekly `fstrim.timer` handles TRIM instead):

```sh
mount -o noatime,compress=zstd,subvol=@ "${SYS}p2" /mnt
mkdir -p /mnt/{.snapshots,var/log,var/cache,var/lib/flatpak,boot/efi,home}
mount -o noatime,compress=zstd,subvol=@snapshots      "${SYS}p2"      /mnt/.snapshots
mount -o noatime,compress=zstd,subvol=@var_log        "${SYS}p2"      /mnt/var/log
mount -o noatime,compress=zstd,subvol=@var_cache      "${SYS}p2"      /mnt/var/cache
mount -o noatime,compress=zstd,subvol=@var_lib_flatpak "${SYS}p2"     /mnt/var/lib/flatpak
mount -o noatime,compress=zstd,subvol=@home           "${HOMEDISK}p1" /mnt/home
mount -o umask=0077                                   "${SYS}p1"      /mnt/boot/efi

findmnt -R /mnt   # verify before continuing
```

## 3. Debootstrap

```sh
debootstrap --arch=amd64 --variant=minbase --include=ca-certificates \
    trixie /mnt "$MIRROR"
```

`minbase` = Essential + apt and *nothing* else — no init, no network stack, no
locales. Every "obvious" package below is therefore mandatory, not decoration.
(`debian-archive-keyring` is an exception you don't have to think about: `apt`
depends on it, so the `Signed-By:` path below is guaranteed to exist.)

**`--include=ca-certificates` is not optional here, it is the fix for a
chicken-and-egg bug.** `ca-certificates` is only a *Recommends* of apt, and
debootstrap doesn't install recommends — so a minbase target has no CA store.
Write https URIs into its sources without it and the very first `apt update`
inside the chroot dies with a certificate verification error, with no working
apt to fix itself. Installing it during the bootstrap closes the loop.
(apt itself needs nothing extra for TLS: the https method has been built into
apt since 1.5; `apt-transport-https` is a transitional dummy package.)

Replace the sources with the deb822 file (Trixie's standard). Three suites,
never anything else — `trixie-updates` **is part of stable** (point-release
fixes), this is not FrankenDebian.

**All four components** are enabled. Enabling a component installs nothing — it
only makes packages *visible* to apt, out of the same archive, the same release
and the same priority (500). That is orthogonal to FrankenDebian, which is about
mixing *suites* and third-party repos.

- `main` — Debian proper.
- `non-free-firmware` — `firmware-amd-graphics` and `amd64-microcode` live
  there; without it the GPU and the CPU microcode update don't load.
- `contrib` — free packages that need non-free bits elsewhere (this is where
  `steam-installer` lives, if you ever switch away from Flatpak Steam).
- `non-free` — nothing in this build uses it today; enabled so that a future
  need (a codec, a driver, a firmware blob outside `non-free-firmware`) is one
  `apt install` away instead of a sources-file edit.

Two things to keep in mind about `non-free` specifically, since it is now
reachable: it is **not** part of Debian proper, and its security support is
best-effort rather than guaranteed by the security team. Nothing changes while
you install nothing from it — and §13's `apt-mark showmanual` audit is what
tells you whether that is still true.

One file, https throughout, using the mirror `nala fetch` picked in §1.1.
`security.debian.org` deliberately stays on its own host — it is dedicated
infrastructure that must always be current, and mirroring it is discouraged.

```sh
rm -f /mnt/etc/apt/sources.list

# NOTE: unquoted heredoc — $MIRROR must expand (it holds the URL from §1.1)
cat > /mnt/etc/apt/sources.list.d/debian.sources <<EOF
Types: deb
URIs: $MIRROR
Suites: trixie trixie-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://security.debian.org/debian-security
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
EOF

cat /mnt/etc/apt/sources.list.d/debian.sources   # verify the URI expanded
```

This stays the **only** file under `/etc/apt/sources.list.d/` for the life of
the system. If you ever re-run `nala fetch` on the installed machine, it will
add `nala-sources.list` alongside it — delete that file and edit the `URIs:`
line here instead, or apt will fetch every suite twice.

## 4. fstab & chroot entry

Generate fstab **before** entering the chroot (the `$SYS`/`$HOMEDISK` variables
exist only out here):

```sh
UUID_EFI=$( blkid -s UUID -o value "${SYS}p1")
UUID_SYSFS=$(blkid -s UUID -o value "${SYS}p2")
UUID_HOMEFS=$(blkid -s UUID -o value "${HOMEDISK}p1")

cat > /mnt/etc/fstab <<EOF
# <filesystem>     <mountpoint>     <type>  <options>                                       <dump> <pass>
UUID=$UUID_SYSFS   /mnt                btrfs   noatime,compress=zstd,subvol=@                  0 0
UUID=$UUID_SYSFS   /mnt/.snapshots      btrfs   noatime,compress=zstd,subvol=@snapshots         0 0
UUID=$UUID_SYSFS   /mnt/var/log         btrfs   noatime,compress=zstd,subvol=@var_log           0 0
UUID=$UUID_SYSFS   /mnt/var/cache       btrfs   noatime,compress=zstd,subvol=@var_cache         0 0
UUID=$UUID_SYSFS   /mnt/var/lib/flatpak btrfs   noatime,compress=zstd,subvol=@var_lib_flatpak   0 0
UUID=$UUID_HOMEFS  /mnt/home            btrfs   noatime,compress=zstd,subvol=@home              0 0
UUID=$UUID_EFI     /mnt/boot/efi        vfat    umask=0077                                      0 1
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
# verify afterwards that /etc/default/keyboard has XKBLAYOUT="us". GNOME reads
# this file for the login screen and the initial input source.)
```

> **No `dpkg --add-architecture i386` here.** Steam comes from Flathub in
> [§11.3](#113-steam-from-flathub--gaming-verification) and brings its own
> 32-bit stack inside the runtime, so the host stays pure amd64 — that removes
> several hundred i386 packages compared to `steam-installer`.

## 6. Package installation

Keep apt's default `Install-Recommends` **on** — under minbase it quietly pulls
plumbing you'd otherwise chase for days.

**nala goes in first**, because everything after it downloads in parallel
instead of one file at a time — that is nala's actual feature, and this section
is where the install spends most of its time:

```sh
apt install -y nala
```

<details>
<summary>Optional: <code>eatmydata</code>, worth a few more minutes on the GNOME group</summary>

`eatmydata` makes dpkg's `fsync()` calls no-ops, which is most of the cost of
unpacking thousands of files. It is an *install-time only* trick: if the machine
loses power mid-transaction the target filesystem is left inconsistent — on a
fresh install that just means starting §3 over, which is why it's acceptable
here and nowhere else.

```sh
apt install -y eatmydata
# then prefix every install below:  eatmydata nala install -y …
# and before leaving the chroot in §8:
#   apt purge -y eatmydata
```
</details>

```sh
# Boot & kernel (no linux-headers: this build has zero DKMS modules —
# AMD GPU support is the in-kernel amdgpu driver)
nala install -y linux-image-amd64 amd64-microcode grub-efi-amd64 efibootmgr \
    dosfstools btrfs-progs firmware-amd-graphics

# Core system — the minbase gaps.
# systemd-sysv = systemd as PID 1 (the "-sysv" suffix only means it ships the
# /sbin/init compat symlinks; this is the standard Debian systemd boot path).
# libpam-systemd + dbus-user-session are load-bearing: logind sessions (GDM /
# seat access) and the D-Bus user bus (pipewire, portals, GNOME Shell).
nala install -y systemd-sysv libpam-systemd dbus dbus-user-session sudo \
    console-setup keyboard-configuration tzdata systemd-timesyncd \
    systemd-resolved ca-certificates curl wget less man-db bash-completion \
    nano pciutils usbutils smartmontools util-linux-extra

# Network (Ethernet only — no wireless firmware, bluetooth never installed).
# No nm-applet: GNOME Shell talks to NetworkManager natively.
nala install -y network-manager

# Security & maintenance (nala itself is already in, from the top of this section)
nala install -y apparmor apparmor-utils ufw gufw unattended-upgrades needrestart

# Snapshots (grub-btrfs itself comes in §9; inotify-tools is its dependency)
nala install -y snapper snapper-gui inotify-tools

# GNOME — gnome-core is Debian's OWN minimal GNOME metapackage (shell, gdm3,
# nautilus, control-center, terminal, keyring, text-editor, loupe, software).
# The three extras are NOT pulled in by gnome-core/gnome-shell in Trixie and
# are load-bearing here:
#   xdg-desktop-portal-gnome → the portal backend Flatpak apps talk to
#                              (file chooser, screenshots, screen sharing)
#   xdg-desktop-portal-gtk   → fallback backend + GTK settings for non-GNOME apps
#   xwayland                 → every X11 client, Steam included
nala install -y gnome-core xdg-desktop-portal-gnome xdg-desktop-portal-gtk xwayland

# Audio
nala install -y pipewire pipewire-pulse pipewire-alsa wireplumber rtkit pavucontrol

# Fonts (GNOME brings Cantarell + Adwaita icons itself)
nala install -y fonts-noto-core fonts-noto-color-emoji fonts-jetbrains-mono

# Flatpak + Flathub integration in GNOME Software (remote is added in §11.2)
nala install -y flatpak gnome-software-plugin-flatpak

# Desktop apps & QoL tools — repo first (Don't Break Debian §D.2)
nala install -y firefox-esr thunderbird mpv gvfs xdg-user-dirs wl-clipboard \
    fastfetch git htop rsync unzip zstd

# Host graphics & performance (Steam itself is a Flatpak — see §11.3)
nala install -y mesa-vulkan-drivers vulkan-tools mesa-utils gamemode \
    zram-tools bpftune power-profiles-daemon profile-sync-daemon
```

**Optional trim (minimal really means minimal).** These three are *Recommends*
of `gnome-core`, not dependencies, so removing them is supported and apt will
not pull them back on upgrade:

```sh
apt purge gnome-tour gnome-user-share gnome-remote-desktop
```

⚠ Run it **without** `-y` and read the removal list first: if `gnome-core`,
`gnome-shell` or `gdm3` appears there, answer **N** and skip this step — that
would mean the Trixie dependency graph differs from what this guide assumed.
Do **not** purge `evolution-data-server` (Shell calendar) or `ibus` (keyboard
input plumbing) — those genuinely break things.

## 7. zram, user, GRUB, graphical target

```sh
# zram: 50% of RAM, zstd
sed -i -e 's/^#\? *ALGO=.*/ALGO=zstd/' -e 's/^#\? *PERCENT=.*/PERCENT=50/' \
    /etc/default/zramswap

cat > /etc/sysctl.d/99-zram.conf <<'EOF'
# zram-friendly VM tuning
vm.swappiness = 180
vm.page-cluster = 0
EOF

# User (root stays locked — sudo-only administration, Debian convention).
# Only the sudo group is needed: logind grants seat/device access (video,
# input, audio) to the active graphical session automatically.
useradd -m -G sudo -s /bin/bash baris
passwd baris

# GRUB (no cryptodisk, no os-prober — single-OS UEFI machine)
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian
sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=3/' /etc/default/grub
update-grub

# Graphical login: gdm3 arrives with gnome-core and registers itself as the
# display manager. Make the boot target explicit and verify the registration —
# under a chroot install this is worth checking rather than assuming.
systemctl set-default graphical.target
ls -l /etc/systemd/system/display-manager.service   # → .../gdm3.service
# If that symlink is missing:
#   systemctl enable gdm3

# Services for first boot
systemctl enable NetworkManager systemd-timesyncd systemd-resolved \
    fstrim.timer zramswap
```

> **No tty1 autologin.** GDM asks for the password, which is what unlocks
> gnome-keyring's login keyring in the same step (saved Wi-Fi/app/browser
> secrets). Autologin would leave the keyring locked and prompt you separately —
> see [C.4](#c4-gnome-session--gdm) if you decide to switch anyway.

## 8. Leave chroot & first boot

Pre-flight checks, still inside the chroot:

```sh
apt policy                      # ONLY trixie / trixie-updates / trixie-security, all https
ls /etc/apt/sources.list.d/     # exactly one file: debian.sources
dpkg --print-foreign-architectures   # must print NOTHING (no i386)
ls /boot/vmlinuz-* /boot/initrd.img-*   # kernel + initramfs exist
efibootmgr -v | grep -i debian  # EFI boot entry present
systemctl get-default           # graphical.target
ls -l /etc/systemd/system/display-manager.service
# apt purge -y eatmydata        # only if you used the optional §6 trick
exit
```

Back in the live shell:

```sh
umount -R /mnt
reboot
```

Remove the USB. The machine should boot to the **GDM login screen**; log in as
`baris` and you land in GNOME on Wayland. Sections §9–§13 run on the installed
system (Terminal: press `Super`, type `terminal`).

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
`Post-Invoke` pair. It fires for apt, nala, unattended-upgrades *and* GNOME
Software's PackageKit backend (they all drive libapt/dpkg). apt may invoke dpkg
several times per transaction; the `/run` marker collapses that into one
pre/post snapshot pair per apt run.

Flatpak transactions are **not** covered by these hooks — Flatpak doesn't touch
dpkg, and `/var/lib/flatpak` is deliberately outside the root snapshots
([§2](#2-partitioning--filesystems)). Flatpak has its own rollback, see
[D.5](#d5-flatpak-upkeep).

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
sudo systemctl daemon-reload               # the unit file is brand new — reload FIRST
sudo systemctl enable --now grub-btrfsd    # watches /.snapshots by default
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

## 11. GNOME, Flatpak, Steam & performance polish

### 11.1 GNOME session basics

```sh
xdg-user-dirs-update          # ~/Downloads, ~/Pictures, ... for Nautilus & apps

# Verify the session is what we think it is
echo "$XDG_SESSION_TYPE"      # → wayland
echo "$XDG_CURRENT_DESKTOP"   # → GNOME
localectl status              # VC Keymap / X11 Layout → us
wpctl status                  # PipeWire: sink + source present
systemctl --user status xdg-desktop-portal xdg-desktop-portal-gnome
```

Taste settings (optional, all reversible `gsettings` keys):

```sh
gsettings set org.gnome.desktop.interface color-scheme 'prefer-dark'
gsettings set org.gnome.desktop.interface monospace-font-name 'JetBrains Mono 11'
gsettings set org.gnome.desktop.session idle-delay 600          # blank after 10 min
gsettings set org.gnome.desktop.screensaver lock-enabled true
```

**Keep a single apt update path.** GNOME Software would otherwise download
system updates in the background, competing with `unattended-upgrades` and the
apt lock. Leave it as an app store (and the Flatpak front-end) instead:

```sh
gsettings set org.gnome.software download-updates false
```

Security updates keep arriving through unattended-upgrades; Flatpak updates
still work from GNOME Software's Updates page on demand.

### 11.2 Flatpak & Flathub

Flatpak is the *only* sanctioned source outside the Debian archive: it does not
touch dpkg, apps are sandboxed, and each app carries its own runtime — so
nothing here can drag the base system out of stable.

```sh
sudo flatpak remote-add --if-not-exists flathub \
    https://dl.flathub.org/repo/flathub.flatpakrepo

flatpak remotes -d          # flathub listed, system-wide
```

Then **log out and back in** once: the session needs to pick up
`/var/lib/flatpak/exports/share` so Flatpak apps show up in the GNOME overview.

Notes that matter here:

- Installs are **system-wide** (`sudo flatpak install …` / GNOME Software's
  default) → everything lands in `/var/lib/flatpak`, which is its own Btrfs
  subvolume and therefore **outside root snapshots** by design.
- `--user` installs land in `~/.local/share/flatpak` instead — those *are*
  inside `@home` snapshots. Pick one and stay consistent; system-wide is the
  default in this build.
- Theming needs no work: GNOME's portal hands the Adwaita/dark preference to
  sandboxed apps automatically.
- Permissions can be inspected/tightened per app:
  ```sh
  flatpak info --show-permissions com.valvesoftware.Steam
  flatpak override --user com.valvesoftware.Steam --nofilesystem=home
  ```
  (`com.github.tchx84.Flatseal` is the GUI for this, itself a Flatpak.)

Rule of thumb, unchanged from [D.2](#d2-adding-software--the-decision-tree-dont-break-debian-forever):
`nala search` first, Flathub second, `/usr/local` third, third-party apt repos
never.

### 11.3 Steam from Flathub & gaming verification

**Before the first launch**, give Steam's data directory its own nested
subvolume — game installs are tens of gigabytes and have no business inside
`@home` snapshots (nested subvolumes are excluded from their parent's snapshots
automatically):

```sh
mkdir -p ~/.var/app
btrfs subvolume create ~/.var/app/com.valvesoftware.Steam
# (If that errors with permission denied: sudo btrfs subvolume create … &&
#  sudo chown baris:baris ~/.var/app/com.valvesoftware.Steam)
```

```sh
flatpak install -y flathub com.valvesoftware.Steam
flatpak run com.valvesoftware.Steam
```

Verification:

```sh
# Host GPU stack
vulkaninfo --summary          # RADV / AMD GPU listed (not llvmpipe)
glxinfo -B                    # renderer = AMD
gamemoded -t                  # gamemode self-test passes

# The stack Steam actually sees (inside the sandbox)
flatpak run --command=vulkaninfo com.valvesoftware.Steam --summary
```

- **gamemode**: the Flatpak talks to the host `gamemoded` over D-Bus. Per-game
  launch options: `gamemoderun %command%`.
- **Extra game library on another path**: sandboxes can't see it until you say
  so — `flatpak override --user com.valvesoftware.Steam --filesystem=/home/baris/Games`
  (and make that a subvolume too, for the same snapshot reason).
- The runtime ships its own 32-bit graphics stack (`org.freedesktop.Platform.GL32`),
  which is why the host needs no i386 multiarch — and why Mesa inside Steam is
  newer than Trixie's, which is the point.

### 11.4 systemd-oomd (works off the zram swap signal)

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

sudo systemctl daemon-reload             # pick up the drop-ins BEFORE starting oomd
sudo systemctl enable --now systemd-oomd
```

### 11.5 Remaining services & tools

```sh
# BPF auto-tuning + power profiles (both from Debian repos; GNOME shows the
# power profile switcher in the system menu once ppd is running)
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

Optional, safe on this machine (Ethernet-only desktop, no service waits for the
network at boot): drop the `network-online.target` wait, usually a few seconds.

```sh
sudo systemctl disable NetworkManager-wait-online.service
# revert any time: sudo systemctl enable NetworkManager-wait-online.service
```

Optional, keeps `@var_log` from growing unbounded (it is outside snapshots, so
nothing else caps it):

```sh
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/00-size.conf >/dev/null <<'EOF'
[Journal]
SystemMaxUse=500M
EOF
sudo systemctl restart systemd-journald
```

## 12. Rollback: procedure & mandatory drill

This build uses the **Arch-style** model: fstab pins `subvol=@`, grub-btrfs
boots snapshots read-only for inspection, and restoring = swapping a snapshot
into `@`. (`snapper rollback` relies on the default-subvolume mechanism and is
**not** used here.)

**What a root rollback does and does not cover:**

| Subvolume | Rolled back with `@`? |
|---|---|
| `@` (system, `/boot`, `/etc`, `/usr`, `/var/lib` except flatpak) | ✅ yes |
| `@var_lib_flatpak` (Flatpak apps/runtimes) | ❌ no — by design, see [D.5](#d5-flatpak-upkeep) |
| `@var_log`, `@var_cache` | ❌ no — deliberate (logs survive the rollback, which is what you want when diagnosing) |
| `@home` (incl. `~/.var/app` Flatpak data) | ❌ no — restore separately from the `home` snapper config if ever needed |

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
ls /etc/apt/sources.list.d/             # exactly one file: debian.sources
grep -c '^URIs: https://' /etc/apt/sources.list.d/debian.sources   # → 2
dpkg --print-foreign-architectures      # empty: no i386 multiarch

# 2. Manual package review — everything here should be recognizable from this guide
apt-mark showmanual | sort

# 3. Only out-of-repo software: grub-btrfs (file list committed to this repo)
cat ~/grub-btrfs-installed-files.txt

# 4. Flatpak side: one remote, apps you chose, nothing else
flatpak remotes -d
flatpak list --app

# 5. Snapshot lifecycle
sudo snapper -c root list        # timeline + apt pre/post pairs accumulating
systemctl list-timers | grep -E 'snapper|fstrim|apt'

# 6. System health
systemctl --failed               # must be empty
sudo aa-status                   # AppArmor enforcing
sudo ufw status verbose          # active, deny incoming
findmnt /var/lib/flatpak         # subvol=@var_lib_flatpak — snapshot isolation intact
```

Cold-boot test: power off, power on → GDM appears, password login lands in
GNOME with network, audio, portals and Flatpak apps working, no manual
intervention.

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

(`/var/lib/flatpak` is not mounted here on purpose — no repair needs it, and
leaving it out keeps the recovery mount list short.)

## Appendix B: What was deliberately left out

| Item | Why it's not here |
|---|---|
| LUKS2 / disk encryption | Design decision (home desktop). Historical note: the original plan (Argon2id + GRUB-unlocked /boot) could never have booted — Trixie's GRUB 2.12 cannot open Argon2id LUKS2; that landed in GRUB 2.14. |
| Secure Boot / sbctl | Disabled in firmware. sbctl isn't in Debian repos and custom keys demand re-signing hooks on every kernel/GRUB update — an unbootable-system trap. |
| sway + waybar/wofi/dunst/foot/grim/slurp/swaylock | Replaced by GNOME. GNOME Shell provides the panel, launcher, notifications, screenshots, lock screen and idle handling itself; `grim`/`slurp` wouldn't even work here (they need wlroots screencopy, which Mutter doesn't implement). |
| `gnome` / `task-gnome-desktop` (full GNOME) | `gnome-core` is Debian's own minimal GNOME metapackage — the same maintainers, the same defaults, without games/office/extras. |
| `network-manager-gnome` (nm-applet) | GNOME Shell has NetworkManager integration built in; a tray applet would be a second, redundant UI. |
| `mate-polkit` / other polkit agents | gnome-shell **is** the polkit agent under GNOME. |
| i386 multiarch + `steam-installer` | Steam comes from Flathub; its runtime carries the 32-bit graphics stack, so the host stays pure amd64 (several hundred fewer packages) and its Mesa is newer than Trixie's. Note: `steam-installer` is in **contrib**, not non-free — that component stays enabled if you ever want to switch back. |
| Separate /boot partition | Would exclude kernels from snapshots → booting old snapshots breaks on kernel/module mismatch. |
| `@tmp` subvolume | Trixie's /tmp is tmpfs by default; keeping Debian defaults is the point. |
| `linux-headers-amd64` | Zero DKMS modules in this build (amdgpu is in-kernel). Install only when a DKMS need appears. |
| `checkinstall` | Abandoned since 2017, pollutes dpkg state — the opposite of Don't Break Debian. |
| low_latency_layer, dmemcg-booster (GitHub tweak scripts) | Unaudited third-party scripts touching sysctl/cgroups. gamemode + zram + bpftune (all from Debian repos) cover the goal. |
| GNOME Shell extensions | Every one of them is a Shell-version-coupled third-party script; they break on the next GNOME. If something is truly needed, prefer the packaged `gnome-shell-extension-*` from the Debian archive. |
| cups, bluetooth, wifi firmware | Hardware/needs don't exist here (Ethernet-only desktop, no printer). Not installed rather than installed-then-disabled. |
| Third-party apt repos, backports, testing/unstable pins | FrankenDebian. If something is missing: Flathub for apps, or wait for the next stable. |

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

**`apt update` in the chroot: "certificate verification failed" / "server
certificate verification failed. CAfile: none"** → the target has no CA store,
i.e. `--include=ca-certificates` was left off the debootstrap line (§3) → don't
hand-edit certificates; either redo debootstrap with the flag, or bridge it once
over http:
```sh
sed -i 's|^URIs: https://|URIs: http://|' /etc/apt/sources.list.d/debian.sources
apt update && apt install -y ca-certificates
sed -i 's|^URIs: http://|URIs: https://|' /etc/apt/sources.list.d/debian.sources
apt update
```

**`Release file for … is not valid yet` / hash sum mismatch / 404 on a package
that definitely exists** → the mirror `nala fetch` picked is lagging behind or
was mid-sync (most likely right after a point release) → point the URI back at
the redirector and move on:
```sh
sed -i 's|^URIs: .*/debian$|URIs: https://deb.debian.org/debian|' \
    /etc/apt/sources.list.d/debian.sources
apt update
```
("not valid yet" can also be a clock problem — check `date` in the live session
before blaming the mirror.)

**`nala fetch` returns few or no mirrors** → `--https-only` plus a country
filter is a narrow intersection → drop the `-c` flags and let it rank globally,
or skip the whole step and use `MIRROR=https://deb.debian.org/debian`.

**nala's output renders as garbage in the chroot / it hangs on a prompt** →
rich TUI meeting a dumb terminal → `apt install …` is a drop-in replacement for
every `nala install …` line in §6; nothing else in the guide depends on nala.

**Inside chroot: `perl: warning: Setting locale failed`** → normal until
`locale-gen` has run; the `export LC_ALL=C.UTF-8` at the top of §5 silences it.
Harmless either way.

**Inside chroot: DNS broken (`Temporary failure resolving deb.debian.org`)** →
`/etc/resolv.conf` was copied as a dangling symlink → exit chroot, re-do the
`rm -f` + `cp --dereference` lines from §4, re-enter.

**tzdata/console-setup ask interactive questions anyway** →
`DEBIAN_FRONTEND=noninteractive` wasn't exported in *this* shell (it does not
survive exiting/re-entering chroot) → re-export, or just answer the prompts.

**`apt install gnome-core` pulls a surprising amount / runs out of space** →
expected size is roughly 2–3 GB for the GNOME group; if the system partition is
tight, check `df -h /mnt` before, not after. `/var/cache` is a separate
subvolume, so `apt clean` frees space without touching snapshots.

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

**Emergency mode: "Dependency failed for /home" (or /.snapshots, /var/lib/flatpak
…)** → fstab typo or wrong UUID → in the emergency shell: `journalctl -xb | grep
-i mount` to see which line, fix `/etc/fstab` (nano works there), `systemctl
daemon-reload`, `exit` or reboot. Cross-check with `blkid`.

### C.4 GNOME session & GDM

**Boots to a text console, GDM never appears** → the display manager isn't
registered or the boot target is wrong → check, in order:
```sh
systemctl get-default                                  # graphical.target
ls -l /etc/systemd/system/display-manager.service      # → gdm3.service
systemctl status gdm3
journalctl -b -u gdm3 --no-pager | tail -40
```
Fix with `sudo systemctl enable gdm3 && sudo systemctl set-default graphical.target`.
Log in on tty2 (`Ctrl+Alt+F2`) to run these.

**GDM shows only an X11 session / `echo $XDG_SESSION_TYPE` says x11** → GDM
disabled Wayland, almost always because the GPU driver didn't come up → check
`journalctl -b | grep -i amdgpu`, confirm `firmware-amd-graphics` is installed,
and make sure `WaylandEnable=false` is *not* set in `/etc/gdm3/daemon.conf`.

**Login loops back to GDM after entering the password** → the user session dies
instantly → `journalctl -b _UID=$(id -u baris) --no-pager | tail -50`. Usual
causes under minbase: `dbus-user-session` or `libpam-systemd` missing (§6), or a
full `/home` (`df -h /home`).

**Everything looks like GNOME but portals/screen-sharing/Flatpak file dialogs
fail** → the portal backend isn't installed or isn't running (it is NOT pulled
in automatically — see §6) →
```sh
systemctl --user status xdg-desktop-portal xdg-desktop-portal-gnome
apt policy xdg-desktop-portal-gnome xdg-desktop-portal-gtk
```
Install what's missing, then log out and back in.

**No sound** → `systemctl --user status pipewire wireplumber pipewire-pulse`;
if "Failed to connect to bus", you're testing over `sudo`/ssh instead of inside
the desktop session. `wpctl status` must list a sink.

**Keyboard layout is not US at the GDM screen** → GDM reads
`/etc/default/keyboard` → verify `XKBLAYOUT="us"`, then
`sudo dpkg-reconfigure keyboard-configuration`. Inside the session it's
Settings → Keyboard → Input Sources instead.

**gufw / snapper-gui never show their admin prompt** → under GNOME the polkit
agent is part of gnome-shell, so this means the app was launched outside the
session (ssh, tty) → launch it from the GNOME overview.

**If you switch to autologin anyway** (`/etc/gdm3/daemon.conf`:
`[daemon] / AutomaticLoginEnable=true / AutomaticLogin=baris`) → expect a
"Unlock login keyring" prompt at every first app that needs a secret. The two
honest options: keep typing the password once per boot, or open
Passwords & Keys (`seahorse`), right-click the *Login* keyring → Change
Password → set an empty one, which stores your saved secrets **unencrypted on
disk**. On an unencrypted disk that is a real, deliberate downgrade — the
password-login default in §7 exists precisely to avoid this fork.

### C.5 Snapper & grub-btrfs

**`create-config` fails: "'.snapshots' already exists"** → you ran it while
`@snapshots` was still mounted → follow §9.1 exactly (umount → rmdir →
create-config → swap subvolumes).

**No pre/post pairs after apt runs** → hook scripts not firing →
`apt-config dump | grep -i invoke` must show both scripts; check they are
executable; run `/usr/local/sbin/snapper-apt-pre.sh` by hand and read the
error. A stale `/run/snapper-apt-pre-num` (from a crashed run) suppresses new
pre-snapshots — delete it. If snapper itself errors about D-Bus in a rescue
environment, `snapper --no-dbus …` works as root.

**GRUB has no "snapshots" submenu** → there were zero snapshots at
`update-grub` time, or grub-btrfsd isn't watching → create one
(`sudo snapper -c root create -d test`), check
`systemctl status grub-btrfsd` (and `sudo systemctl daemon-reload` if the unit
is "not found" right after `make install`), run `sudo update-grub` again.

**Booted a snapshot and "everything is broken/read-only"** → expected:
snapshot boots are read-only inspection environments; services that need to
write will fail. Look around, confirm it's the state you want, then do the
[§12 restore](#12-rollback-procedure--mandatory-drill) — don't try to "use"
a snapshot boot.

**After a restore the GRUB menu looks stale / lists wrong kernels** → the menu
is generated from the *current* `@`'s `/boot` → `sudo update-grub` after every
restore (it's step 4 of the procedure for exactly this reason).

**After a restore, a Flatpak app is broken or "from the future"** →
`/var/lib/flatpak` is outside root snapshots by design (§12 table) → fix it on
the Flatpak side: `flatpak update`, or roll a single app back with
`flatpak update --commit=<hash> <app>` (see [D.5](#d5-flatpak-upkeep)).

**`btrfs subvolume delete /mnt/@broken` refuses** → you created a nested
subvolume inside `@` at some point → `btrfs subvolume list /mnt` and delete
children first. In this layout `@` has no nested subvolumes by design.

### C.6 Flatpak

**Flatpak apps don't appear in the GNOME overview** → the session hasn't picked
up `/var/lib/flatpak/exports/share` → log out and back in (a full reboot is
never required, but the *session* must restart).

**`flatpak install` fails: "No remote refs found" / TLS errors** → remote not
added system-wide, or DNS is broken → `flatpak remotes -d`, then
`resolvectl query dl.flathub.org`.

**An app can't see your files** → sandbox, working as intended →
`flatpak override --user <app-id> --filesystem=/path` (or per-file access via
the portal file chooser, which needs no override at all).

**GNOME Software shows no Flathub apps** → `gnome-software-plugin-flatpak`
missing, or GNOME Software was running when the remote was added → install the
plugin (§6), then `pkill gnome-software` and reopen it.

**Disk usage climbing** → old runtimes pile up →
`flatpak uninstall --unused` and `flatpak repair` (as root for system installs).
`sudo btrfs filesystem usage /` — remember `/var/lib/flatpak` shares the system
partition even though it's a separate subvolume.

### C.7 Steam & gaming

**`vulkaninfo` shows only `llvmpipe` on the host** → GPU firmware/driver not
active → `dmesg | grep -i amdgpu` (look for firmware load errors),
`apt reinstall firmware-amd-graphics`, reboot.

**Steam starts but games fail with missing 32-bit Vulkan/GL** → the runtime's
32-bit extension didn't install → `flatpak update com.valvesoftware.Steam`, and
verify with
`flatpak list --runtime | grep -i gl32`. The host has (and needs) no i386
packages — do not "fix" this with `dpkg --add-architecture i386`.

**`gamemoderun` has no effect / D-Bus errors in the game log** → host
`gamemoded` isn't running (`systemctl --user status gamemoded`) or the launch
option was set on the wrong game. Test the host side first with `gamemoded -t`.

**Games install into `~/.var/app/...` and blow up home snapshots** → the nested
subvolume from §11.3 wasn't created before first launch → close Steam, move the
directory aside, create the subvolume, move the contents back:
```sh
mv ~/.var/app/com.valvesoftware.Steam{,.old}
btrfs subvolume create ~/.var/app/com.valvesoftware.Steam
rsync -aAX --remove-source-files ~/.var/app/com.valvesoftware.Steam.old/ \
      ~/.var/app/com.valvesoftware.Steam/
rm -rf ~/.var/app/com.valvesoftware.Steam.old
```

**Controller not detected** → the Steam Flatpak ships the udev rules it needs,
but they only apply after a replug; if it persists, check
`flatpak info --show-permissions com.valvesoftware.Steam` for `device=all`.

## Appendix D: Living with the system — maintenance & upgrades

### D.1 Routine

```sh
sudo nala upgrade          # interactive system updates (snapshots happen automatically)
sudo nala autoremove       # after big removals
flatpak update             # applications
systemctl --failed         # occasionally; must stay empty
```

If downloads ever feel slow, re-benchmark mirrors — but keep the one-file rule:
`sudo nala fetch --debian trixie --https-only --non-free --auto --fetches 3`,
then copy the winning URL into the `URIs:` line of
`/etc/apt/sources.list.d/debian.sources` and **delete** the
`nala-sources.list` it wrote, so apt doesn't index every suite twice.

Security updates install themselves (unattended-upgrades) and are snapshot-
bracketed by the §9.2 hooks. `needrestart` tells you when a reboot is due
(kernel updates). Flatpak updates are a separate, independent stream — that
separation is the feature.

### D.2 Adding software — the decision tree (Don't Break Debian, forever)

1. **Debian repo first**: `nala search <thing>`. If it's there, done.
2. **Flathub second** (GUI apps not in trixie, or too old to be useful):
   ```sh
   flatpak search <thing>
   flatpak install flathub <app-id>
   ```
   The remote is already configured (§11.2) and portals are installed, so
   sandboxed apps integrate with GNOME out of the box.
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

### D.5 Flatpak upkeep

Flatpak is outside dpkg *and* outside the snapshots, so it carries its own
maintenance and its own rollback:

```sh
flatpak update                       # apps + runtimes
flatpak uninstall --unused           # orphaned runtimes (the usual space hog)
sudo flatpak repair                  # verify/repair the system installation
flatpak list --app --columns=application,version,size
```

Per-app rollback (the Flatpak equivalent of a snapshot restore):

```sh
flatpak remote-info --log flathub com.valvesoftware.Steam   # list commits
sudo flatpak update --commit=<hash> com.valvesoftware.Steam
```

Keep the app list itself reproducible — it is not covered by any snapshot:

```sh
flatpak list --app --columns=application > ~/flatpak-apps.txt   # commit this
```

### D.6 grub-btrfs updates

Occasionally (a few times a year):
```sh
cd ~/grub-btrfs && git fetch --tags
git checkout "$(git describe --tags --abbrev=0)"
sudo make install 2>&1 | tee ~/grub-btrfs-installed-files.txt
sudo systemctl daemon-reload && sudo systemctl restart grub-btrfsd
```

### D.7 Point releases and the Debian 14 upgrade

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
   regenerates (`sudo update-grub`); if it broke, reinstall per D.6.
7. Flatpak needs nothing during the release upgrade — apps keep running on
   their own runtimes. That is exactly why user applications live there.
8. Keep the snapshot until you've lived on Forky for a week or two.

### D.8 When something feels wrong

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
