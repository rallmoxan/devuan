# Step 3: Snapper, grub-btrfs, Hardening & Tuning

## Objective
First boot into the new system: wire up snapshots + apt hooks + grub-btrfs,
enable the security baseline, apply the (audited, minimal) performance tweaks.

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
Wrap in small scripts under `/usr/local/sbin/`. This fires for apt, nala AND
unattended-upgrades (all drive libapt/dpkg).

### 3. grub-btrfs (from GitHub — not in Debian repos)
1. Clone https://github.com/Antynea/grub-btrfs, checkout the latest release TAG
   (not master).
2. `make install` — then record the exact installed file list in this repo's notes
   (it writes to `/etc/grub.d/41_snapshots-btrfs`, `/etc/default/grub-btrfs/`,
   systemd units) so it can be removed cleanly. This is the ONLY out-of-repo
   software on the system.
3. Enable `grub-btrfsd.service` watching `/.snapshots` (snapper mode).
4. Test: create a snapshot → `update-grub` → snapshot submenu appears.

### 4. Rollback procedure (Arch-style — DOCUMENT it, then DRILL it once)
`snapper rollback` is NOT used (fstab pins subvol=@). Restore procedure:
1. Boot the desired read-only snapshot from the GRUB "snapshots" submenu, verify it.
2. From that boot (or live USB): mount the toplevel
   (`mount -o subvolid=5 /dev/nvme0n1p2 /mnt`).
3. `mv /mnt/@ /mnt/@broken` then
   `btrfs subvol snapshot /mnt/.snapshots/<N>/snapshot /mnt/@` (rw copy).
4. Reboot into restored `@`; delete `@broken` after verifying.
Drill this once on the fresh install (break something trivial, restore it) BEFORE
trusting it. Keep the live USB around.

### 5. Firewall & security baseline
- ufw: `default deny incoming`, `default allow outgoing`, `ufw enable`
  (service enabled at boot). gufw available for GUI.
- AppArmor: verify `aa-status` shows profiles enforced (installed in step 2).
- unattended-upgrades: enable via `/etc/apt/apt.conf.d/20auto-upgrades`
  (Update-Package-Lists + Unattended-Upgrade daily); origins: `,-security` (default)
  + optionally trixie-updates. Verify with `unattended-upgrade --dry-run --debug`.

### 6. Services & tuning (audited, minimal — no third-party scripts)
- Enable: `bpftune power-profiles-daemon systemd-oomd`.
  systemd-oomd: enable `ManagedOOMSwap=kill` / `ManagedOOMMemoryPressure=kill` on
  user/system slices via drop-ins (zram provides the swap signal it needs).
- systemd-resolved: enabled in step 2; point NetworkManager at it
  (`dns=systemd-resolved`) and verify `resolvectl status`.
- profile-sync-daemon (per-user, as baris): `systemctl --user enable psd.service`;
  if using overlayfs mode, add the required sudoers line per psd docs.
- Firefox on SSD/psd: profile lives in RAM via psd; optionally
  `browser.cache.disk.enable=false` in about:config.
- TRIM: `fstrim.timer` enabled in step 2 — verify with `fstrim -av` once.
- Boot review: `systemd-analyze blame` / `critical-chain` — investigate outliers,
  do NOT bulk-disable units.
- NOT installed, nothing to disable: cups, bluetooth. Do not install "just in case".
- Dropped by design review (do not resurrect): checkinstall, low_latency_layer,
  dmemcg-booster.

### 7. Gaming verification
- Steam first run as user (steam-installer bootstraps the client).
- `vulkaninfo --summary` shows the AMD GPU (RADV); `glxinfo -B` via mesa-utils.
- gamemode: `gamemoded -t` passes.

### 8. Final verification (Don't Break Debian audit)
- `apt policy`: ONLY trixie, trixie-updates, trixie-security (all priority 500),
  no other origins.
- `apt list --installed | grep -v trixie` style spot-check → the only non-repo
  software on the system is grub-btrfs (file list documented).
- Snapshot lifecycle: apt transaction creates pre/post pair; timeline+cleanup
  timers active; grub-btrfs regenerates the snapshot menu automatically.
- Reboot test: cold boot lands in sway (autologin) with network, audio, portals
  (screenshot via grim, screen share test) working.
