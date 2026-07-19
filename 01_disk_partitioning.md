# Step 1: Disk Partitioning & Btrfs Layout (no LUKS, no LVM)

## Objective
From the **Debian Live USB**, partition both NVMe disks and create the Btrfs
subvolume layout, mounted and ready for debootstrap.

## ⚠ Preconditions (verify before touching anything)
1. Booted from Debian Live (Trixie) — NOT from CachyOS.
2. `lsblk -o NAME,SIZE,MODEL,SERIAL` — confirm which device is which.
   **Both disks will be completely erased.** Double-check nvme0n1/nvme1n1 naming;
   device order can differ between boots. Set variables once, use them everywhere:
   ```
   SYS=/dev/nvme0n1        # system disk (will hold ESP + root btrfs)
   HOMEDISK=/dev/nvme1n1   # home disk  (will hold home btrfs)
   ```
   (Never name this variable `HOME` — that would clobber the shell's own $HOME.)
3. UEFI mode confirmed: `[ -d /sys/firmware/efi ]`.
4. Secure Boot already disabled in firmware setup.

## Instructions for Claude Code

### 1. Partition tables (GPT)
- `$SYS`:
  - p1: 1 GiB, type EFI System → ESP
  - p2: remaining space, type Linux filesystem → Btrfs
- `$HOMEDISK`:
  - p1: whole disk, type Linux filesystem → Btrfs

### 2. Filesystems
- `mkfs.vfat -F32 -n EFI  ${SYS}p1`
- `mkfs.btrfs -L system   ${SYS}p2`
- `mkfs.btrfs -L home     ${HOMEDISK}p1`

### 3. Subvolumes
On `system` (mount ${SYS}p2 at /mnt temporarily, create, then umount):
- `@`           → will be `/`  (contains `/boot` — intentional: snapshots include kernels)
- `@snapshots`  → will be `/.snapshots`
- `@var_log`    → will be `/var/log`
- `@var_cache`  → will be `/var/cache` (keeps apt cache out of snapshots)

On `home` (same dance with ${HOMEDISK}p1):
- `@home`       → will be `/home`

Notes:
- **No `@tmp`** — Trixie mounts /tmp as tmpfs by default; keep it.
- **No separate /boot partition** — /boot stays inside `@`.

### 4. Final mount (target = /mnt)
Mount options everywhere: `noatime,compress=zstd,subvol=<name>`
(no `discard` — weekly `fstrim.timer` is enabled in step 3 instead).

```
mount -o noatime,compress=zstd,subvol=@          ${SYS}p2  /mnt
mkdir -p /mnt/{.snapshots,var/log,var/cache,boot/efi,home}
mount -o noatime,compress=zstd,subvol=@snapshots ${SYS}p2  /mnt/.snapshots
mount -o noatime,compress=zstd,subvol=@var_log   ${SYS}p2  /mnt/var/log
mount -o noatime,compress=zstd,subvol=@var_cache ${SYS}p2  /mnt/var/cache
mount -o noatime,compress=zstd,subvol=@home      ${HOMEDISK}p1 /mnt/home
mount -o umask=0077                              ${SYS}p1      /mnt/boot/efi
```

### 5. Verification before step 2
- `findmnt -R /mnt` matches the table above.
- `btrfs subvolume list /mnt` shows all 5 subvolumes.
- No LVM, no LUKS anywhere (`lsblk` shows plain partitions).

## Target layout summary

| Device | FS | Subvol | Mountpoint | Options |
|---|---|---|---|---|
| nvme0n1p1 | FAT32 | — | /boot/efi | umask=0077 |
| nvme0n1p2 | Btrfs | @ | / | noatime,compress=zstd |
| nvme0n1p2 | Btrfs | @snapshots | /.snapshots | noatime,compress=zstd |
| nvme0n1p2 | Btrfs | @var_log | /var/log | noatime,compress=zstd |
| nvme0n1p2 | Btrfs | @var_cache | /var/cache | noatime,compress=zstd |
| nvme1n1p1 | Btrfs | @home | /home | noatime,compress=zstd |

(zram swap is configured post-install in step 2 — nothing to do on-disk here.)
