# Step 3: Snapper, grub-btrfs, Flatpak, Hardening & Tuning

## Objective
First boot into the new system (GDM → GNOME): wire up snapshots + apt hooks +
grub-btrfs, add Flathub, enable the security baseline, apply the (audited, minimal)
performance tweaks.

## Instructions for Claude Code

### 1. Snapper (root + home)
1. **Order matters** — fstab already mounts `@snapshots` at `/.snapshots`, and
   create-config refuses to run if `/.snapshots` exists. Sequence:
   `umount /.snapshots && rmdir /.snapshots` →
   `snapper -c root create-config /` →
   `btrfs subvol delete /.snapshots && mkdir /.snapshots && mount /.snapshots && chmod 750 /.snapshots`
2. `snapper -c home create-config /home` — keep the nested `/home/.snapshots`
   subvolume it creates (home snapshots live on the home disk; nested subvolumes
   are auto-excluded from parent snapshots). Nothing to fix.
3. Config tuning (`/etc/snapper/configs/root`): `ALLOW_USERS="baris"`,
   `TIMELINE_CREATE="yes"`, sane limits (e.g. `TIMELINE_LIMIT_HOURLY=5`,
   `DAILY=7`, `WEEKLY=2`, `MONTHLY=0`, `YEARLY=0`, `NUMBER_LIMIT=20`).
   Home config: timeline on, smaller limits.
4. Enable `snapper-timeline.timer` and `snapper-cleanup.timer`.

### 2. APT/nala transaction snapshots
Debian's snapper ships NO apt hook — create `/etc/apt/apt.conf.d/80snapper`:
- `DPkg::Pre-Invoke`  → `snapper -c root create -t pre  -p -d "apt"` (store number in /run)
- `DPkg::Post-Invoke` → `snapper -c root create -t post --pre-number $(cat /run/...) -d "apt"`
Wrap in small scripts under `/usr/local/sbin/`. This fires for apt, nala,
unattended-upgrades AND GNOME Software's PackageKit backend (all drive libapt/dpkg).

Flatpak transactions are **not** covered (no dpkg involved, and `/var/lib/flatpak` is
outside root snapshots by design) — Flatpak has its own rollback, see §4.

### 3. grub-btrfs (from GitHub — not in Debian repos)
1. Clone https://github.com/Antynea/grub-btrfs, checkout the latest release TAG
   (not master).
2. `make install` — then record the exact installed file list in this repo's notes
   (it writes to `/etc/grub.d/41_snapshots-btrfs`, `/etc/default/grub-btrfs/`,
   systemd units) so it can be removed cleanly. This is the ONLY out-of-repo
   software on the system.
3. `systemctl daemon-reload` **first** (the unit is brand new), then enable
   `grub-btrfsd.service` watching `/.snapshots` (snapper mode).
4. Test: create a snapshot → `update-grub` → snapshot submenu appears.

### 4. Rollback procedure (Arch-style — DOCUMENT it, then DRILL it once)
`snapper rollback` is NOT used (fstab pins subvol=@). Restore procedure:
1. Boot the desired read-only snapshot from the GRUB "snapshots" submenu, verify it.
2. From that boot (or live USB): mount the toplevel
   (`mount -o subvolid=5 /dev/nvme0n1p2 /mnt`).
3. `mv /mnt/@ /mnt/@broken` then
   `btrfs subvol snapshot /mnt/.snapshots/<N>/snapshot /mnt/@` (rw copy).
4. Reboot into restored `@`; `update-grub`; delete `@broken` after verifying.

Scope: this restores `@` only. `@var_lib_flatpak`, `@var_log`, `@var_cache` and
`@home` stay as they are — deliberate. Flatpak apps roll back on their own track:
`flatpak remote-info --log flathub <app>` → `flatpak update --commit=<hash> <app>`.

Drill this once on the fresh install (break something trivial, restore it) BEFORE
trusting it. Keep the live USB around.

### 5. Flatpak & Flathub
1. `flatpak remote-add --if-not-exists flathub https://dl.flathub.org/repo/flathub.flatpakrepo`
   as root → **system-wide** remote; installs land in `/var/lib/flatpak` (own subvolume).
2. Log out / log back in once so the session picks up
   `/var/lib/flatpak/exports/share` (Flatpak apps appear in the GNOME overview).
3. Verify: `flatpak remotes -d`; GNOME Software lists Flathub apps
   (`gnome-software-plugin-flatpak` installed in step 2).
4. Stay consistent: system-wide installs only (`--user` installs land in
   `~/.local/share/flatpak`, i.e. *inside* home snapshots).
5. Keep the app list reproducible: `flatpak list --app --columns=application`
   committed to this repo.

### 6. Steam (Flathub) & gaming
1. **Before first launch**, nested subvolume for game data:
   `mkdir -p ~/.var/app && btrfs subvolume create ~/.var/app/com.valvesoftware.Steam`
   (keeps tens of GB out of `@home` snapshots).
2. `flatpak install -y flathub com.valvesoftware.Steam`
3. Verify: host `vulkaninfo --summary` (RADV, not llvmpipe), `glxinfo -B`,
   `gamemoded -t`; inside the sandbox
   `flatpak run --command=vulkaninfo com.valvesoftware.Steam --summary`.
4. gamemode per game: launch option `gamemoderun %command%` (the Flatpak talks to the
   host gamemoded over D-Bus).
5. External library path → `flatpak override --user com.valvesoftware.Steam
   --filesystem=/home/baris/Games` (make that a subvolume too).
6. Do **not** "fix" 32-bit issues with `dpkg --add-architecture i386` — the runtime's
   `org.freedesktop.Platform.GL32` extension is the 32-bit stack.

### 7. Firewall & security baseline
- ufw: `default deny incoming`, `default allow outgoing`, `ufw enable`
  (service enabled at boot). gufw available for GUI.
- AppArmor: verify `aa-status` shows profiles enforced (installed in step 2).
- unattended-upgrades: enable via `/etc/apt/apt.conf.d/20auto-upgrades`
  (Update-Package-Lists + Unattended-Upgrade daily); origins: `,-security` (default)
  + optionally trixie-updates. Verify with `unattended-upgrade --dry-run --debug`.

### 8. Services & tuning (audited, minimal — no third-party scripts)
- Enable: `bpftune power-profiles-daemon systemd-oomd`.
  systemd-oomd: `ManagedOOMSwap=kill` / `ManagedOOMMemoryPressure=kill` drop-ins on
  `-.slice` / `user@.service` (zram provides the swap signal it needs).
  **`systemctl daemon-reload` before `enable --now`** so the drop-ins are live.
- systemd-resolved: enabled in step 2; point NetworkManager at it
  (`dns=systemd-resolved`) and verify `resolvectl status`.
- profile-sync-daemon (per-user, as baris): `systemctl --user enable --now psd`;
  if using overlayfs mode, add the required sudoers line per psd docs.
- GNOME Software: `gsettings set org.gnome.software download-updates false` — keeps
  apt on a single update path (nala + unattended-upgrades); it stays the Flatpak
  front-end.
- TRIM: `fstrim.timer` enabled in step 2 — verify with `fstrim -av` once.
- Boot review: `systemd-analyze blame` / `critical-chain` — investigate outliers,
  do NOT bulk-disable units. Optional and safe here:
  `systemctl disable NetworkManager-wait-online.service`.
- Optional log cap (`/var/log` is outside snapshots, nothing else bounds it):
  `SystemMaxUse=500M` drop-in for journald.
- NOT installed, nothing to disable: cups, bluetooth. Do not install "just in case".
- Dropped by design review (do not resurrect): checkinstall, low_latency_layer,
  dmemcg-booster, GNOME Shell extensions from extensions.gnome.org.

### 9. GNOME session verification
- `echo $XDG_SESSION_TYPE` → `wayland`; `echo $XDG_CURRENT_DESKTOP` → `GNOME`.
- `systemctl --user status xdg-desktop-portal xdg-desktop-portal-gnome` → running
  (screen sharing, Flatpak file chooser depend on it).
- `wpctl status` lists sink + source; `localectl status` shows `us`.
- `xdg-user-dirs-update` once, so Nautilus/apps get ~/Downloads etc.

### 10. Final verification (Don't Break Debian audit)
- `apt policy`: ONLY trixie, trixie-updates, trixie-security (all priority 500),
  all over https, no other origins. `dpkg --print-foreign-architectures` → empty.
- `ls /etc/apt/sources.list.d/` → exactly one file (`debian.sources`). If a later
  `nala fetch` ever adds `nala-sources.list`, move the URL into debian.sources and
  delete it — two files means every suite is indexed twice.
- `apt-mark showmanual | sort` → everything recognizable from step 2.
- The only non-repo software on the system is grub-btrfs (file list documented);
  the only non-dpkg software is Flatpak apps (`flatpak list --app`).
- Snapshot lifecycle: apt transaction creates pre/post pair; timeline+cleanup
  timers active; grub-btrfs regenerates the snapshot menu automatically.
- `findmnt /var/lib/flatpak` → `subvol=@var_lib_flatpak` (snapshot isolation intact).
- Reboot test: cold boot → GDM → password → GNOME with network, audio, portals,
  Flatpak apps working.
