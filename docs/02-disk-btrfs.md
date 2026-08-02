# Aşama 2 — Disk Bölümleme ve BTRFS Subvolume Mimarisi

> ## 🛑 VERİ KAYBI UYARISI
> Bu aşamadaki her komut **geri dönüşsüzdür**. Aşağıdaki aygıt adları
> **örnektir** — kendi sisteminizdekiyle birebir aynı olduğunu varsaymayın.

---

## 2.0 Hedef düzen

```
/dev/nvme1n1  (KÖK DİSKİ)
├── nvme1n1p1   1 GiB   EF00  FAT32   -> /boot/efi
└── nvme1n1p2   kalan   8300  BTRFS   -> etiket: DEVUAN_ROOT
                                          ├── @          -> /
                                          ├── @snapshots -> /.snapshots
                                          └── @var_log   -> /var/log

/dev/nvme0n1  (HOME DİSKİ)
└── nvme0n1p1   tüm disk 8300 BTRFS   -> etiket: DEVUAN_HOME
                                          └── @home     -> /home
```

---

## 2.1 Diskleri kesin olarak tanımlayın

```bash
lsblk -o NAME,SIZE,TYPE,MODEL,SERIAL,MOUNTPOINTS
ls -l /dev/disk/by-id/ | grep nvme
```

Doğru diskleri belirledikten **sonra** değişkenleri ayarlayın. Rehberin geri
kalanı bu iki değişkeni kullanır:

```bash
# ⚠️ AŞAĞIDAKİ İKİ SATIRI KENDİ SİSTEMİNİZE GÖRE DÜZENLEYİN ⚠️
export ROOT_DISK="/dev/nvme1n1"     # üzerine EFI + kök BTRFS kurulacak disk
export HOME_DISK="/dev/nvme0n1"     # yalnızca /home için kullanılacak disk

# NVMe bölüm adları p ekiyle üretilir (nvme1n1 -> nvme1n1p1).
# SATA disk (/dev/sda) kullanıyorsanız aşağıdaki "p" öneki KALDIRILMALIDIR.
export EFI_PART="${ROOT_DISK}p1"
export ROOT_PART="${ROOT_DISK}p2"
export HOME_PART="${HOME_DISK}p1"

echo "EFI=$EFI_PART  ROOT=$ROOT_PART  HOME=$HOME_PART"
```

Son bir kontrol — yanlış diski silmediğinizden emin olun:

```bash
lsblk -o NAME,SIZE,MODEL "$ROOT_DISK" "$HOME_DISK"
```

> **Tek diskli kurulum yapacaksanız:** `HOME_DISK` ile ilgili adımları atlayın,
> `@home` subvolume'ünü kök BTRFS'i içinde oluşturun ve fstab'daki `/home`
> satırında `DEVUAN_HOME` yerine `DEVUAN_ROOT` UUID'sini kullanın. Rehberin geri
> kalanı değişmez.

---

## 2.2 Eski imzaların temizlenmesi ve GPT tabloları

```bash
# ⚠️ Bu komutlar $ROOT_DISK ve $HOME_DISK üzerindeki HER ŞEYİ siler.
wipefs -a "$ROOT_DISK"
wipefs -a "$HOME_DISK"

sgdisk --zap-all "$ROOT_DISK"
sgdisk --zap-all "$HOME_DISK"
```

Kök diski — EFI + kök bölümü:

```bash
sgdisk -n 1:0:+1G   -t 1:ef00 -c 1:"EFI System"     "$ROOT_DISK"
sgdisk -n 2:0:0     -t 2:8300 -c 2:"Devuan Root"    "$ROOT_DISK"

# Home diski — tek bölüm
sgdisk -n 1:0:0     -t 1:8300 -c 1:"Devuan Home"    "$HOME_DISK"

partprobe "$ROOT_DISK" "$HOME_DISK"
sleep 2
lsblk "$ROOT_DISK" "$HOME_DISK"
```

`-n 1:0:+1G` → 1 GiB EFI. Debian/Devuan'da her çekirdek `/boot`'a yazılır ama
`/boot` BTRFS üzerindedir; EFI bölümüne yalnızca GRUB'ın EFI binary'si gider.
1 GiB fazlasıyla yeterlidir (512 MiB de olur).

---

## 2.3 Dosya sistemlerinin oluşturulması

```bash
mkfs.vfat -F 32 -n EFI "$EFI_PART"

mkfs.btrfs -f -L DEVUAN_ROOT "$ROOT_PART"
mkfs.btrfs -f -L DEVUAN_HOME "$HOME_PART"

blkid "$EFI_PART" "$ROOT_PART" "$HOME_PART"
```

> `btrfs-progs` 6.14 `free-space-tree` (space_cache v2) özelliğini varsayılan
> olarak açar. Mount seçeneklerine `space_cache=v2` yine de yazacağız — etkisi
> yok ama düzeni açıkça belgeliyor.

---

## 2.4 Subvolume'lerin oluşturulması

Subvolume'leri oluşturmak için dosya sistemlerini geçici olarak **üst seviyeden**
(subvolid=5) bağlıyoruz.

```bash
mkdir -p /mnt/btrfs-root /mnt/btrfs-home
mount "$ROOT_PART" /mnt/btrfs-root
mount "$HOME_PART" /mnt/btrfs-home

# Kök diskindeki subvolume'ler
btrfs subvolume create /mnt/btrfs-root/@
btrfs subvolume create /mnt/btrfs-root/@snapshots
btrfs subvolume create /mnt/btrfs-root/@var_log

# Home diskindeki subvolume
btrfs subvolume create /mnt/btrfs-home/@home

btrfs subvolume list /mnt/btrfs-root
btrfs subvolume list /mnt/btrfs-home

umount /mnt/btrfs-root /mnt/btrfs-home
rmdir  /mnt/btrfs-root /mnt/btrfs-home
```

### Neden `@var_log` ayrı?

Kök snapshot'ına geri döndüğünüzde günlüklerin de geriye sarmasını istemezsiniz —
sorunun sebebini anlatan kayıtlar tam da o günlüklerdedir. Ayrı subvolume, `@`
snapshot'ına dahil olmaz.

---

## 2.5 Mount seçenekleri

Rehber boyunca kullanılacak ortak seçenekler:

```bash
export BTRFS_OPTS="noatime,compress=zstd:1,space_cache=v2,discard=async"
```

| Seçenek | Neden |
|---|---|
| `noatime` | Okuma başına metadata yazımını keser; SSD ömrü ve CoW gürültüsü için |
| `compress=zstd:1` | En düşük gecikmeli zstd seviyesi; NVMe'de CPU'yu darboğaz yapmaz |
| `space_cache=v2` | Serbest alan ağacı (yeni mkfs'te zaten varsayılan) |
| `discard=async` | **fstrim.timer yerine geçen mekanizma** — aşağıya bakın |
| `subvol=@…` | Her satırda ilgili subvolume |

### TRIM hakkında

`fstrim.timer` bir systemd birimidir; bu sistemde yok. Yerine BTRFS'in kendi
zamanuyumsuz discard'ı kullanılır. btrfs belgelerinden:

> `discard, discard=sync, discard=async, nodiscard`
> *(default: async when devices support it since 6.2, async support since: 5.6)*

Kuracağımız çekirdek 6.12 LTS olduğundan `discard=async` **zaten varsayılan**.
Yine de fstab'a açıkça yazıyoruz. Aşama 6'da `btrfsmaintenance` içindeki
`BTRFS_TRIM_PERIOD` `"none"` bırakılacak ki periyodik `fstrim` ile çakışmasın.

---

## 2.6 Hedef sistemin bağlanması

```bash
# 1) Kök subvolume
mount -o "${BTRFS_OPTS},subvol=@" "$ROOT_PART" /mnt

# 2) Alt dizinler (kök bağlandıktan sonra oluşturulmalı)
mkdir -p /mnt/home /mnt/var/log /mnt/boot/efi

# 3) /home — AYRI DİSK
mount -o "${BTRFS_OPTS},subvol=@home" "$HOME_PART" /mnt/home

# 4) /var/log
mount -o "${BTRFS_OPTS},subvol=@var_log" "$ROOT_PART" /mnt/var/log

# 5) EFI
mount -o umask=0077 "$EFI_PART" /mnt/boot/efi
```

### ⚠️ `@snapshots` şimdi bağlanmıyor — bilerek

`snapper -c root create-config /` komutu, `/.snapshots` **zaten varsa**
(dizin ya da subvolume, fark etmez) hata verip çıkar. Bu yüzden:

- **Şimdi:** `@snapshots` bağlanmaz, `/mnt/.snapshots` oluşturulmaz.
- **Aşama 6'da:** snapper kendi `.snapshots` subvolume'ünü oluşturur, biz onu
  silip yerine `@snapshots`'ı bağlarız ve fstab'a o zaman ekleriz.

---

## 2.7 Kontrol

```bash
findmnt -R /mnt
```

Beklenen (özet):

```
TARGET          SOURCE                              FSTYPE OPTIONS
/mnt            /dev/nvme1n1p2[/@]                  btrfs  rw,noatime,compress=zstd:1,...
├─/mnt/home     /dev/nvme0n1p1[/@home]              btrfs  rw,noatime,compress=zstd:1,...
├─/mnt/var/log  /dev/nvme1n1p2[/@var_log]           btrfs  rw,noatime,compress=zstd:1,...
└─/mnt/boot/efi /dev/nvme1n1p1                      vfat   rw,...
```

UUID'leri şimdiden kaydedin (Aşama 5'te fstab için gerekecek):

```bash
cat >> /root/devuan-env.sh <<EOF
export ROOT_DISK="$ROOT_DISK"
export HOME_DISK="$HOME_DISK"
export EFI_PART="$EFI_PART"
export ROOT_PART="$ROOT_PART"
export HOME_PART="$HOME_PART"
export BTRFS_OPTS="$BTRFS_OPTS"
export UUID_EFI="$(blkid -s UUID -o value "$EFI_PART")"
export UUID_ROOT="$(blkid -s UUID -o value "$ROOT_PART")"
export UUID_HOME="$(blkid -s UUID -o value "$HOME_PART")"
EOF

source /root/devuan-env.sh
echo "EFI=$UUID_EFI"; echo "ROOT=$UUID_ROOT"; echo "HOME=$UUID_HOME"
```

---

➡️ Sonraki: [Aşama 3 — Debootstrap ile Devuan Excalibur İndirme](03-debootstrap.md)
