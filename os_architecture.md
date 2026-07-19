# BarzbugOS Architecture (Target: Debian 13 Trixie)

> Revised after design review (2026-07-19). Decisions: no disk encryption, Secure Boot
> disabled in firmware, sway desktop, Steam from official repos, Arch-style rollback.

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
  - Components: `main contrib non-free non-free-firmware`
    (`non-free` is required for `steam-installer`; it is an official Debian component.)
  - Foreign arch: `i386` enabled (Steam / 32-bit Vulkan).
  - Format: deb822 (`/etc/apt/sources.list.d/debian.sources`), Trixie's modern default.
- **Hardware profile:** Desktop, AMD CPU + AMD GPU (amdgpu/mesa), **Ethernet only**
  (no WiFi, no Bluetooth → no wireless firmware; bluetooth never installed).

## 2. Storage Layout (no LUKS, no LVM — plain Btrfs)
- **nvme0n1** (system):
  - p1: 1 GB, FAT32, ESP → `/boot/efi`
  - p2: rest, Btrfs → subvolumes `@`, `@snapshots`, `@var_log`, `@var_cache`
  - `/boot` lives **inside `@`** (no separate /boot partition) → snapshots contain
    kernel + initramfs → snapshot boots and rollbacks are fully consistent.
- **nvme1n1** (home):
  - p1: whole disk, Btrfs → subvolume `@home` → `/home`
- **Mount options:** `noatime,compress=zstd` (no `discard` — TRIM comes from weekly
  `fstrim.timer` instead).
- **No `@tmp`:** Trixie mounts `/tmp` as tmpfs by default — we keep the Debian default.
- **Swap:** zram only (zram-tools, 50% RAM, zstd). No hibernation (desktop, accepted).

## 3. Boot
- **Bootloader:** GRUB (grub-efi-amd64) + grub-btrfs (from GitHub — not packaged in
  Debian; installed files tracked for clean removal).
- **Secure Boot:** DISABLED in firmware. sbctl plan dropped (not in Debian repos,
  would require manual signing hooks on every kernel/GRUB update).
- **Login:** tty1 autologin → `exec sway` from shell profile.
  ⚠ Accepted trade-off: no disk encryption + autologin = zero protection against
  physical access. Conscious decision for a home desktop.

## 4. Snapshots & Rollback
- **Snapper** configs: `root` (nvme0n1) and `home` (nvme1n1), cleanup timers enabled.
- **Apt integration:** custom `DPkg::Pre-Invoke`/`Post-Invoke` hooks (Debian's snapper
  ships no apt hook). Works for `nala` too (nala drives libapt → hooks fire).
- **Rollback style: Arch-style** — fstab pins `subvol=@`; grub-btrfs boots old
  snapshots read-only; restore = manual snapshot-to-`@` swap (procedure documented in
  step 3). `snapper rollback` (default-subvolume mechanism) is NOT used.

## 5. Desktop (sway, full experience)
- **Core:** sway, swaybg, swaylock, swayidle, waybar, wofi, dunst, foot (terminal),
  xwayland.
- **Plumbing:** pipewire (+pipewire-pulse, wireplumber, rtkit), xdg-desktop-portal-wlr
  + xdg-desktop-portal-gtk, a polkit agent (mate-polkit), grim+slurp+wl-clipboard.
- **Apps:** firefox-esr, thunderbird, Thunar (+gvfs, tumbler), imv, mpv, pavucontrol,
  gufw, snapper-gui, fastfetch.
- **Network:** NetworkManager + nm-applet (Debian binary package:
  `network-manager-gnome`) in waybar tray.

## 6. Gaming & Performance
- Steam via `steam-installer` (non-free) + i386 multiarch + `mesa-vulkan-drivers:i386`.
- gamemode, mesa-vulkan-drivers, vulkan-tools, mesa-utils.
- zram-tools (50%, zstd) + `vm.swappiness=180`, `vm.page-cluster=0`.
- bpftune — **from Debian repo** (packaged in Trixie, no source build needed).
- power-profiles-daemon, profile-sync-daemon (per-user).

## 7. Security & Maintenance
- AppArmor enabled (Debian default — must be installed explicitly under minbase).
- ufw: default deny incoming, enabled at boot. gufw for GUI.
- unattended-upgrades: security (+ trixie-updates) origins, configured, verified.
- systemd-oomd + zram; systemd-timesyncd; systemd-resolved (optional, see step 3).
- Package management: `nala` for interactive use (apt remains the backend — there is
  no "set as default frontend" mechanism; unattended-upgrades always uses apt).

## 8. Explicitly Dropped (design review, 2026-07-19)
| Item | Reason |
|---|---|
| LUKS2 / Argon2id / crypttab | User decision: no encryption. (Also: Trixie's GRUB 2.12 cannot open Argon2id LUKS2 — Argon2 support only landed in GRUB 2.14.) |
| Secure Boot / sbctl | User decision. sbctl is not in Debian repos; custom-key maintenance burden (re-sign on every update) not worth it. |
| Separate encrypted /boot | Obsolete without LUKS; /boot inside `@` gives full-rollback consistency. |
| `@tmp` subvolume | Trixie defaults /tmp to tmpfs; keep the default (Don't Break Debian). |
| `checkinstall` | Abandoned upstream (2017), pollutes dpkg state; nothing left that needs a source build besides grub-btrfs. |
| low_latency_layer, dmemcg-booster | Unaudited third-party scripts — highest "break Debian" risk in the old plan. Skipped entirely; gamemode+zram+bpftune cover the goal. |
| `linux-headers-amd64` | No DKMS modules anywhere in this build (AMD GPU = in-kernel amdgpu). Reinstall only if a DKMS need appears. |
| cups / bluetooth "disable" tweaks | Neither gets installed in the first place (minbase); nothing to disable. |
