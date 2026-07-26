# BarzbugOS Architecture (Target: Debian 13 Trixie)

> Revised after design review (2026-07-19) and the desktop revision (2026-07-26).
> Decisions: no disk encryption, Secure Boot disabled in firmware, **minimal GNOME
> (`gnome-core`) with GDM password login**, **Steam and other non-archive apps via
> Flatpak/Flathub**, Arch-style rollback.

## 1. System Foundation
- **Distro:** Debian 13 (Trixie, stable). Installed via `debootstrap --variant=minbase`
  from a **Debian Live USB** (not from the running CachyOS).
- **Init:** systemd as PID 1 — installed via `systemd-sysv` (the standard Debian
  package that makes systemd the init; the "-sysv" suffix only means it provides the
  `/sbin/init` compatibility symlinks — no sysvinit involved).
- **Language:** English system — locale `en_US.UTF-8` only, **US keyboard layout**.
  Timezone: Europe/Istanbul.
- **Repo Policy — No FrankenDebian:**
  - Suites: `trixie`, `trixie-updates`, `trixie-security` — and NOTHING else.
    (`trixie-updates` is part of stable, not FrankenDebian. No backports, no testing,
    no unstable, no third-party apt repos.)
  - Components: **all four** — `main contrib non-free non-free-firmware`.
    Enabling a component installs nothing; it only makes packages visible, from the
    same archive/suite/priority. Orthogonal to FrankenDebian (which is about mixing
    suites and third-party repos), so there is no stability cost to keeping the full
    set available for future needs.
    - `non-free-firmware`: `firmware-amd-graphics`, `amd64-microcode` (in use).
    - `contrib`: unused today; this is where `steam-installer` lives if the Flatpak
      decision is ever reversed.
    - `non-free`: unused today; enabled deliberately (user decision, 2026-07-26) so a
      future codec/driver/blob need doesn't require a sources edit. Caveat to
      remember: non-free is not part of Debian proper and its security support is
      best-effort — which matters only once something is actually installed from it.
      The `apt-mark showmanual` audit is what surfaces that.
  - Foreign arch: **none**. i386 multiarch was dropped together with `steam-installer`;
    Flatpak Steam ships its own 32-bit stack inside the runtime.
  - Format: deb822 (`/etc/apt/sources.list.d/debian.sources`), Trixie's modern default.
    Exactly **one** sources file, for the life of the system — `nala fetch` is used as
    a mirror *discovery* tool only, never as a second sources file (duplicate suites).
  - Transport: **https** for both the mirror and security.debian.org. Rationale is
    privacy, not integrity (apt's OpenPGP signatures already cover integrity); the
    cost is that `ca-certificates` must exist in the target *before* the first
    `apt update` → `debootstrap --include=ca-certificates`.
  - Mirror: chosen by `nala fetch --https-only` in the live session, before
    debootstrap, so the bootstrap and the ~2–3 GB of §6 packages all come from it.
    `https://deb.debian.org/debian` (CDN redirector, always in sync) is the
    documented fallback whenever a picked mirror lags.
- **Application sources (in priority order):** Debian archive → Flathub (system-wide
  Flatpak) → `/usr/local` → nothing else, ever.
- **Hardware profile:** Desktop, AMD CPU + AMD GPU (amdgpu/mesa), **Ethernet only**
  (no WiFi, no Bluetooth → no wireless firmware; bluetooth never installed).

## 2. Storage Layout (no LUKS, no LVM — plain Btrfs)
- **nvme0n1** (system):
  - p1: 1 GB, FAT32, ESP → `/boot/efi`
  - p2: rest, Btrfs → subvolumes `@`, `@snapshots`, `@var_log`, `@var_cache`,
    `@var_lib_flatpak`
  - `/boot` lives **inside `@`** (no separate /boot partition) → snapshots contain
    kernel + initramfs → snapshot boots and rollbacks are fully consistent.
  - `@var_lib_flatpak` → `/var/lib/flatpak`: multi-GB app/runtime storage that updates
    on its own schedule. Excluding it keeps root snapshots small and keeps a system
    rollback from dragging applications back in time. Flatpak has its own per-app
    rollback (`flatpak update --commit=`).
- **nvme1n1** (home):
  - p1: whole disk, Btrfs → subvolume `@home` → `/home`
  - Per-app nested subvolumes for bulk data (e.g.
    `~/.var/app/com.valvesoftware.Steam`, `~/Games`) — nested subvolumes are excluded
    from `@home` snapshots automatically.
- **Mount options:** `noatime,compress=zstd` (no `discard` — TRIM comes from weekly
  `fstrim.timer` instead).
- **No `@tmp`:** Trixie mounts `/tmp` as tmpfs by default — we keep the Debian default.
- **Swap:** zram only (zram-tools, 50% RAM, zstd). No hibernation (desktop, accepted).

## 3. Boot
- **Bootloader:** GRUB (grub-efi-amd64) + grub-btrfs (from GitHub — not packaged in
  Debian; installed files tracked for clean removal).
- **Secure Boot:** DISABLED in firmware. sbctl plan dropped (not in Debian repos,
  would require manual signing hooks on every kernel/GRUB update).
- **Login:** GDM3 (arrives with `gnome-core`), **password login, no autologin**.
  Rationale: the login password unlocks gnome-keyring's login keyring in the same
  step. Autologin would leave saved secrets locked behind a second prompt, and the
  only way around that is an unencrypted keyring — a bad trade on an unencrypted disk.
  ⚠ Still accepted: no disk encryption means zero protection against physical access;
  GDM protects the running session only.

## 4. Snapshots & Rollback
- **Snapper** configs: `root` (nvme0n1) and `home` (nvme1n1), cleanup timers enabled.
- **Apt integration:** custom `DPkg::Pre-Invoke`/`Post-Invoke` hooks (Debian's snapper
  ships no apt hook). Works for `nala`, unattended-upgrades and GNOME Software's
  PackageKit backend too (all drive libapt → hooks fire).
- **Rollback style: Arch-style** — fstab pins `subvol=@`; grub-btrfs boots old
  snapshots read-only; restore = manual snapshot-to-`@` swap (procedure documented in
  step 3). `snapper rollback` (default-subvolume mechanism) is NOT used.
- **Scope:** a root rollback restores `@` only. `@var_lib_flatpak`, `@var_log`,
  `@var_cache` and `@home` are deliberately outside it — applications, logs and user
  data are managed on their own tracks.

## 5. Desktop (minimal GNOME)
- **Core:** `gnome-core` — Debian's own minimal GNOME metapackage (GNOME 48):
  gnome-shell, gdm3, nautilus, gnome-control-center, terminal, gnome-keyring,
  gnome-text-editor, loupe, gnome-disk-utility, gnome-software. Not `gnome` and not
  `task-gnome-desktop` (games/office/extras).
- **Explicit additions** (verified *not* pulled in by gnome-core/gnome-shell in Trixie,
  and load-bearing here): `xdg-desktop-portal-gnome` (portal backend — Flatpak file
  chooser, screenshots, screen sharing), `xdg-desktop-portal-gtk` (fallback backend),
  `xwayland` (all X11 clients, Steam included).
- **Session:** Wayland (GDM default with amdgpu).
- **Audio:** pipewire, pipewire-pulse, pipewire-alsa, wireplumber, rtkit, pavucontrol.
- **Not installed, on purpose:** nm-applet (`network-manager-gnome`) — GNOME Shell has
  NetworkManager built in; a separate polkit agent — gnome-shell *is* the polkit agent;
  a notification daemon, launcher, panel, screenshot or lock tool — all part of Shell.
- **Optional trim:** `gnome-tour`, `gnome-user-share`, `gnome-remote-desktop` are
  *Recommends* of gnome-core and can be purged safely.
- **Apps (repo):** firefox-esr, thunderbird, mpv, gvfs, fastfetch, wl-clipboard, gufw,
  snapper-gui + CLI tooling (git, htop, rsync, unzip, zstd, nala).
- **Apps (Flathub):** anything the archive doesn't ship or ships too old — Steam first
  and foremost.
- **GNOME Software:** kept as the Flatpak front-end; `download-updates` turned off so
  apt updates stay on a single path (nala + unattended-upgrades).

## 6. Flatpak / Flathub
- `flatpak` + `gnome-software-plugin-flatpak` from the Debian archive.
- One remote, added **system-wide**: `flathub`. Installs go to `/var/lib/flatpak`
  (own subvolume, §2).
- Sandboxed, dpkg-independent, versioned separately from the OS → this is the
  sanctioned answer to "the archive doesn't have it / it's too old", and the reason no
  third-party apt repo is ever needed.
- Portals are installed explicitly (§5), so file chooser, screen sharing and theming
  (Adwaita/dark via portal settings) work for sandboxed apps without extra config.

## 7. Gaming & Performance
- **Steam:** `com.valvesoftware.Steam` from Flathub. No i386 multiarch on the host, no
  `steam-installer`; the runtime carries its own 32-bit graphics stack
  (`org.freedesktop.Platform.GL32`) with a newer Mesa than Trixie ships.
  Game data lives in a nested subvolume (`~/.var/app/com.valvesoftware.Steam`), out of
  home snapshots.
- Host graphics/verification: mesa-vulkan-drivers, vulkan-tools, mesa-utils.
- gamemode (host daemon; Flatpak Steam calls it over D-Bus via `gamemoderun %command%`).
- zram-tools (50%, zstd) + `vm.swappiness=180`, `vm.page-cluster=0`.
- bpftune — **from Debian repo** (packaged in Trixie, no source build needed).
- power-profiles-daemon (GNOME exposes it in the system menu), profile-sync-daemon
  (per-user).
- systemd-oomd with drop-ins on `-.slice` / `user@.service` (zram provides the swap
  signal it needs).

## 8. Security & Maintenance
- AppArmor enabled (Debian default — must be installed explicitly under minbase).
- ufw: default deny incoming, enabled at boot. gufw for GUI.
- unattended-upgrades: security (+ trixie-updates) origins, configured, verified.
- systemd-timesyncd; systemd-resolved with NetworkManager pointed at it.
- Package management: `nala` for interactive use (apt remains the backend — there is
  no "set as default frontend" mechanism; unattended-upgrades always uses apt).
- Flatpak maintenance is a separate track: `flatpak update`,
  `flatpak uninstall --unused`, app list committed to this repo.

## 9. Explicitly Dropped
| Item | Reason |
|---|---|
| LUKS2 / Argon2id / crypttab | User decision: no encryption. (Also: Trixie's GRUB 2.12 cannot open Argon2id LUKS2 — Argon2 support only landed in GRUB 2.14.) |
| Secure Boot / sbctl | User decision. sbctl is not in Debian repos; custom-key maintenance burden (re-sign on every update) not worth it. |
| sway + waybar/wofi/dunst/foot/grim/slurp/swaylock/swayidle | Superseded by GNOME (2026-07-26). Shell covers panel, launcher, notifications, screenshots, lock and idle. `grim`/`slurp` are wlroots-only and would not even function under Mutter. |
| `gnome` / `task-gnome-desktop` | Full GNOME with games/office/extras — the opposite of "minimal base". `gnome-core` is the Debian-maintained minimal target. |
| `network-manager-gnome` (nm-applet), mate-polkit | Redundant under GNOME Shell. |
| i386 multiarch + `steam-installer` | Replaced by Flatpak Steam: no multiarch sprawl, newer Mesa in the runtime, sandboxed, independently rollback-able. Correction to the previous revision: `steam-installer` is in **contrib**, not `non-free`. |
| Separate encrypted /boot | Obsolete without LUKS; /boot inside `@` gives full-rollback consistency. |
| `@tmp` subvolume | Trixie defaults /tmp to tmpfs; keep the default (Don't Break Debian). |
| `checkinstall` | Abandoned upstream (2017), pollutes dpkg state; nothing left that needs a source build besides grub-btrfs. |
| low_latency_layer, dmemcg-booster | Unaudited third-party scripts — highest "break Debian" risk in the old plan. Skipped entirely; gamemode+zram+bpftune cover the goal. |
| GNOME Shell extensions (extensions.gnome.org) | Third-party scripts coupled to the Shell version; they break on every GNOME update. Packaged `gnome-shell-extension-*` from the archive only, if ever. |
| `linux-headers-amd64` | No DKMS modules anywhere in this build (AMD GPU = in-kernel amdgpu). Reinstall only if a DKMS need appears. |
| cups / bluetooth "disable" tweaks | Neither gets installed in the first place (minbase); nothing to disable. |
