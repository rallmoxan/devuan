# Step 2: Base System Installation (Debootstrap)

## Objective
Deploy Debian 13 Trixie onto /mnt (prepared in step 1), configure it in chroot,
and make it boot on its own: GRUB, network, user, GNOME + GDM.

## Instructions for Claude Code

### 0. Mirror selection (in the live session, BEFORE debootstrap)
```
apt install -y nala ca-certificates
nala fetch --debian trixie --https-only --non-free --auto --fetches 3 -c TR -c DE -c NL
MIRROR=$(awk '/^deb /{print $2; exit}' /etc/apt/sources.list.d/nala-sources.list)
MIRROR=${MIRROR:-https://deb.debian.org/debian}
```
`nala fetch` is used here **only to discover** the fastest https mirror; its
`nala-sources.list` belongs to the throwaway live session. The installed system
gets one hand-written deb822 file (step 2) — never two overlapping sources files.
Fallback whenever a mirror lags or misbehaves: `https://deb.debian.org/debian`
(a CDN redirector, always in sync).

### 1. Bootstrap
```
debootstrap --arch=amd64 --variant=minbase --include=ca-certificates \
    trixie /mnt "$MIRROR"
```
Reminder: minbase = Essential + apt ONLY. No init, no network stack, no locales —
everything below is mandatory, not optional. (`debian-archive-keyring` is the one
freebie: `apt` depends on it, so the `Signed-By:` path below always exists.)

`--include=ca-certificates` is **required** for the https sources in step 2:
ca-certificates is only a *Recommends* of apt and debootstrap skips recommends,
so without it the first `apt update` in the chroot fails certificate
verification with no working apt left to repair itself. (apt needs nothing else
for TLS — the https method is built in; `apt-transport-https` is a transitional
dummy.)

### 2. APT sources (deb822 — Trixie standard)
Remove any `/mnt/etc/apt/sources.list` debootstrap left behind. Create
`/mnt/etc/apt/sources.list.d/debian.sources`:
```
Types: deb
URIs: $MIRROR                      # from step 0 — https, expand it (unquoted heredoc)
Suites: trixie trixie-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://security.debian.org/debian-security
Suites: trixie-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```
https everywhere (transport privacy — apt's OpenPGP signatures already provide
integrity). `security.debian.org` keeps its own host: dedicated infrastructure
that must always be current, mirroring it is discouraged.
Policy check: these three suites and nothing else, ever (No FrankenDebian).
**All four components** are enabled — a component only makes packages visible,
from the same archive and suite, so this costs nothing in stability terms:
- `non-free-firmware` → firmware-amd-graphics, amd64-microcode (in use).
- `contrib` → unused today (`steam-installer` lives there if the Flatpak decision
  is ever reversed).
- `non-free` → unused today; enabled so a future codec/driver/blob need is one
  `apt install` away. Not part of Debian proper; security support is best-effort,
  which only starts to matter once something is installed from it.

### 3. Chroot
Bind /dev, /proc, /sys, /run (and efivarfs if needed), copy the *contents* of
/etc/resolv.conf (it is a symlink into /run on the live image), then chroot.
All following steps run inside.

### 4. Identity & basics
- **No `dpkg --add-architecture i386`** — Steam is a Flatpak now; the host stays
  pure amd64.
- `apt update`
- hostname: `barzbug` (+ matching `/etc/hosts`: `127.0.1.1 barzbug`)
- locales: `en_US.UTF-8` ONLY (English system), set as default
- timezone: `Europe/Istanbul`
- keyboard: **US layout** (`XKBLAYOUT="us"` — Debian default, verify in
  `/etc/default/keyboard`; GDM and the GNOME login screen read this file)

### 5. fstab
Build from `blkid` UUIDs, mirroring the step-1 table exactly
(`subvol=@...` pinned — required by the Arch-style rollback model), including the
`/var/lib/flatpak` → `subvol=@var_lib_flatpak` line. ESP line: `umask=0077`.
`/tmp`: NO entry (Trixie tmpfs default).
Verify with `findmnt --verify --tab-file /mnt/etc/fstab`.

### 6. Packages (grouped; single nala transaction per group)

**Install `nala` first** (`apt install -y nala`), then use `nala install -y` for
every group below — parallel downloads are the whole point and this section is
where the install spends its time. Plain `apt install` is a drop-in fallback if
nala's TUI misbehaves in the chroot.
Optional install-time speedup: `apt install -y eatmydata` and prefix each line
with `eatmydata` (dpkg fsync becomes a no-op; power loss mid-install means
redoing step 1, hence install-time only — purge it before step 12).

**Boot & kernel:**
`linux-image-amd64 amd64-microcode grub-efi-amd64 efibootmgr dosfstools btrfs-progs firmware-amd-graphics`
(No linux-headers: no DKMS in this build.)

**Core system (minbase gaps):**
`systemd-sysv libpam-systemd dbus dbus-user-session sudo console-setup keyboard-configuration tzdata systemd-timesyncd systemd-resolved ca-certificates curl wget less man-db bash-completion nano pciutils usbutils smartmontools util-linux-extra`
- `systemd-sysv` = **systemd as PID 1** (the "-sysv" suffix only means it ships the
  `/sbin/init` compat symlinks — this is how every standard Debian systemd install
  boots; no sysvinit anywhere).
- `libpam-systemd` + `dbus-user-session` are load-bearing under minbase: logind user
  sessions (GDM, seat access) and the D-Bus user bus (pipewire, portals, Shell) need
  them.

**Network (Ethernet only — no wireless firmware, no bluetooth):**
`network-manager`
(No `network-manager-gnome`: GNOME Shell talks to NetworkManager natively.)

**Security & maintenance:**
`apparmor apparmor-utils ufw gufw unattended-upgrades needrestart`
(nala is already installed — see the note above.)

**Snapshots:**
`snapper snapper-gui inotify-tools`
(inotify-tools = grub-btrfsd dependency; grub-btrfs itself is installed in step 3.)

**GNOME (minimal):**
`gnome-core xdg-desktop-portal-gnome xdg-desktop-portal-gtk xwayland`
- `gnome-core` = Debian's own minimal GNOME metapackage (shell, gdm3, nautilus,
  control-center, terminal, keyring, text-editor, loupe, disk-utility, software).
- The other three are **not** pulled in by gnome-core/gnome-shell in Trixie and are
  load-bearing: GNOME portal backend (Flatpak file chooser, screenshots, screen
  sharing), GTK portal fallback, and Xwayland for every X11 client (Steam included).
- Optional trim (all *Recommends*, safe to purge, run without `-y` and read the list):
  `gnome-tour gnome-user-share gnome-remote-desktop`. Never purge
  `evolution-data-server` (Shell calendar) or `ibus`.

**Audio:**
`pipewire pipewire-pulse pipewire-alsa wireplumber rtkit pavucontrol`

**Fonts:**
`fonts-noto-core fonts-noto-color-emoji fonts-jetbrains-mono`
(GNOME brings Cantarell + adwaita-icon-theme itself.)

**Flatpak:**
`flatpak gnome-software-plugin-flatpak`
(The Flathub remote is added on the installed system — step 3.)

**Desktop apps (repo first):**
`firefox-esr thunderbird mpv gvfs xdg-user-dirs wl-clipboard fastfetch git htop rsync unzip zstd`

**Host graphics & performance:**
`mesa-vulkan-drivers vulkan-tools mesa-utils gamemode zram-tools bpftune power-profiles-daemon profile-sync-daemon`
(No `mesa-vulkan-drivers:i386`, no `steam-installer` — Flatpak Steam ships its own
32-bit stack.)

### 7. zram
`/etc/default/zramswap`: `ALGO=zstd`, `PERCENT=50`.
`/etc/sysctl.d/99-zram.conf`:
```
vm.swappiness = 180
vm.page-cluster = 0
```

### 8. User
- `useradd -m -G sudo -s /bin/bash baris` + password.
  (Only `sudo` — logind grants the active graphical session its device access;
  `video`/`input` group membership is not needed under GNOME/GDM.)
- Root account: locked (sudo-only administration, Debian convention).

### 9. GRUB
- `grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=debian`
- `update-grub`
- No cryptodisk, no os-prober needed (single-OS machine).

### 10. Graphical login (GDM3, password — no autologin)
- `systemctl set-default graphical.target`
- Verify gdm3 registered itself:
  `ls -l /etc/systemd/system/display-manager.service` → `gdm3.service`
  (fallback: `systemctl enable gdm3`).
- **No tty1 autologin, no `.bash_profile` session launcher.** The GDM password is
  what unlocks gnome-keyring's login keyring; autologin would leave saved secrets
  behind a second prompt (or force an unencrypted keyring).

### 11. Enable services (systemctl enable, effective on first boot)
`NetworkManager systemd-timesyncd systemd-resolved fstrim.timer zramswap`
(ufw + the rest in step 3.)

### 12. Exit checklist before reboot
- `apt policy` inside chroot: only trixie / trixie-updates / trixie-security, all https.
- `ls /etc/apt/sources.list.d/` → exactly one file (`debian.sources`).
- `apt purge -y eatmydata` if the optional speedup was used.
- `dpkg --print-foreign-architectures` → empty (no i386).
- fstab UUIDs verified against blkid; `findmnt --verify` clean.
- EFI boot entry exists (`efibootmgr -v` shows "debian").
- initramfs exists for the installed kernel (`ls /boot/initrd.img-*`).
- `systemctl get-default` → graphical.target; display-manager symlink present.
