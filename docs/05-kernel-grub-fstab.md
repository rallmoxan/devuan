# Aşama 5 — Çekirdek, Bootloader ve BTRFS Yapılandırması

Chroot içinde devam ediyoruz.

---

## 5.1 `/etc/fstab`

`@snapshots` bu aşamada **bilinçli olarak eklenmez** — snapper'ın
`create-config` komutu `/.snapshots` varsa çalışmaz. Aşama 6'da eklenecek.

```bash
UUID_ROOT=$(findmnt -no UUID -T /)
UUID_HOME=$(findmnt -no UUID -T /home)
UUID_EFI=$(findmnt -no UUID -T /boot/efi)

printf 'ROOT=%s\nHOME=%s\nEFI=%s\n' "$UUID_ROOT" "$UUID_HOME" "$UUID_EFI"

# Üçü de dolu mu? Boş bir UUID açılmayan bir sistem demektir.
for v in UUID_ROOT UUID_HOME UUID_EFI; do
    [ -n "${!v}" ] || echo "!!! $v BOŞ — devam etmeyin, mount düzenini kontrol edin !!!"
done
```

Üç UUID de dolu geldiyse fstab'ı yazın:

```bash
OPTS="noatime,compress=zstd:1,space_cache=v2,discard=async"

cat > /etc/fstab <<EOF
# <dosya sistemi>                            <bağlama noktası>  <tip>   <seçenekler>                 <dump> <pass>

# Kök — BTRFS @ subvolume (nvme1n1)
UUID=$UUID_ROOT   /           btrfs   ${OPTS},subvol=@          0      0

# Ayrı disk — BTRFS @home subvolume (nvme0n1)
UUID=$UUID_HOME   /home       btrfs   ${OPTS},subvol=@home      0      0

# Günlükler — snapshot geri alımından etkilenmemesi için ayrı subvolume
UUID=$UUID_ROOT   /var/log    btrfs   ${OPTS},subvol=@var_log   0      0

# EFI System Partition
UUID=$UUID_EFI    /boot/efi   vfat    umask=0077,shortname=winnt 0     2

# NOT: /.snapshots satırı Aşama 6'da, snapper yapılandırıldıktan SONRA eklenecek.
EOF

cat /etc/fstab
```

### Neden BTRFS satırlarında `pass` = 0?

BTRFS'in açılışta çalışan bir `fsck`'i yoktur; bütünlük denetimi `btrfs scrub`
ile yapılır (Aşama 6'da `btrfsmaintenance` bunu aylık zamanlar). `pass` alanına
0'dan farklı bir değer yazmak açılışta gereksiz gecikme/uyarı üretir.

### `subvolid=` KULLANMAYIN

`genfstab -U` gibi araçlar bazen `subvolid=257` benzeri satırlar üretir. Bir
snapshot'a geri döndüğünüzde subvolume ID değişir; `subvolid` sabitlenmişse
sistem açılmaz. Her zaman **isimle** (`subvol=@`) bağlayın.

---

## 5.2 Firmware ve mikrokod

```bash
apt install -y firmware-linux firmware-misc-nonfree

# ⚠️ İŞLEMCİNİZE GÖRE BİRİNİ SEÇİN
apt install -y intel-microcode     # Intel CPU
# apt install -y amd64-microcode   # AMD CPU

# Şüphedeyseniz:
grep -m1 'vendor_id' /proc/cpuinfo
```

Kablosuz yongaya göre ek firmware gerekebilir (`firmware-iwlwifi`,
`firmware-realtek`, `firmware-atheros`, `firmware-amd-graphics` …).

---

## 5.3 İki yedekli LTS çekirdek

Debian 13 / Devuan 6'nın çekirdeği **6.12 LTS**'tir. Arşivde aynı anda birden
fazla 6.12.x nokta sürümü bulunur; iki farklı sürümü yan yana kurmak, biri
açılmadığında GRUB'ın "Advanced options" menüsünden diğerine düşebilmenizi
sağlar.

### Arşivdeki 6.12 LTS çekirdeklerini listeleyin

```bash
apt update
apt-cache search --names-only '^linux-image-6\.12\.[0-9]+\+deb13-amd64$' \
  | awk '{print $1}' | sort -V
```

2026-08-02 itibarıyla dönen sonuç:

```
linux-image-6.12.86+deb13-amd64
linux-image-6.12.94+deb13-amd64
```

### Kurulum

```bash
mapfile -t K < <(apt-cache search --names-only '^linux-image-6\.12\.[0-9]+\+deb13-amd64$' \
                  | awk '{print $1}' | sort -V)
echo "Bulunan çekirdek sayısı: ${#K[@]}"

KERNEL_MAIN="linux-image-amd64"          # metapaket -> en yeni 6.12.x
KERNEL_BACKUP="${K[-2]:-}"               # bir önceki nokta sürümü

echo "Ana:   $KERNEL_MAIN"
echo "Yedek: ${KERNEL_BACKUP:-<yok>}"

apt install -y "$KERNEL_MAIN" linux-headers-amd64 initramfs-tools

if [ -n "$KERNEL_BACKUP" ]; then
    apt install -y "$KERNEL_BACKUP" "${KERNEL_BACKUP/image/headers}"
    # autoremove'un yedek çekirdeği silmesini engelle
    apt-mark manual "$KERNEL_BACKUP"
fi
```

### Arşivde tek bir 6.12.x varsa

Yedek olarak `excalibur-backports` içindeki daha yeni LTS çekirdeğini kullanın
(2026-08-02'de 6.16.12):

```bash
cat > /etc/apt/sources.list.d/backports.list <<'EOF'
deb http://deb.devuan.org/merged excalibur-backports main contrib non-free non-free-firmware
EOF

# Backports varsayılan olarak düşük öncelikli olsun; yalnızca istediğimiz gelsin
cat > /etc/apt/preferences.d/backports.pref <<'EOF'
Package: *
Pin: release n=excalibur-backports
Pin-Priority: 100
EOF

apt update
apt-cache search --names-only '^linux-image-6\.[0-9]+\.[0-9]+\+deb13-amd64$' | sort -V | tail -3
# ⚠️ Aşağıdaki paket adını yukarıdaki çıktıya göre güncelleyin
apt install -y -t excalibur-backports linux-image-6.16.12+deb13-amd64
apt-mark manual linux-image-6.16.12+deb13-amd64
```

### Kurulu çekirdekleri doğrulayın

```bash
dpkg -l | grep -E '^ii\s+linux-image' | awk '{print $2, $3}'
ls -1 /boot/vmlinuz-* /boot/initrd.img-*
```

İki `vmlinuz-*` ve iki `initrd.img-*` görmelisiniz.

---

## 5.4 initramfs ayarları

```bash
# zstd sıkıştırma: daha küçük initrd, daha hızlı açılım
sed -i 's/^COMPRESS=.*/COMPRESS=zstd/' /etc/initramfs-tools/initramfs.conf
grep -q '^COMPRESS=' /etc/initramfs-tools/initramfs.conf || echo 'COMPRESS=zstd' >> /etc/initramfs-tools/initramfs.conf

# Donanım değişikliklerine dayanıklı olsun (harici disk, farklı denetleyici vs.)
sed -i 's/^MODULES=.*/MODULES=most/' /etc/initramfs-tools/initramfs.conf

# BTRFS ve zram modüllerini garantiye al
cat >> /etc/initramfs-tools/modules <<'EOF'
btrfs
zstd
EOF

# Tüm çekirdekler için initramfs'i yeniden üret
update-initramfs -u -k all

ls -lh /boot/initrd.img-*
```

---

## 5.5 GRUB (UEFI) kurulumu

```bash
apt install -y grub-efi-amd64 efibootmgr os-prober

# Secure Boot AÇIKSA ek olarak:
# apt install -y shim-signed grub-efi-amd64-signed
```

> **Secure Boot notu:** Devuan, Debian'ın imzalı GRUB/shim ikililerini merged
> depo üzerinden dağıtır. Çoğu makinede çalışır; sorun yaşarsanız en hızlı
> çözüm UEFI ayarlarından Secure Boot'u kapatmaktır. Bu rehberin geri kalanı
> Secure Boot kapalı varsayar.

```bash
grub-install \
    --target=x86_64-efi \
    --efi-directory=/boot/efi \
    --bootloader-id=Devuan \
    --recheck

ls -la /boot/efi/EFI/Devuan/
efibootmgr -v | grep -i devuan
```

`grub-install`, `/boot/efi/EFI/Devuan/grubx64.efi` dosyasını yazar ve UEFI
NVRAM'ine bir önyükleme girdisi ekler.

---

## 5.6 `/etc/default/grub`

```bash
cat > /etc/default/grub <<'EOF'
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_TIMEOUT_STYLE=menu
GRUB_DISTRIBUTOR="Devuan"

# "quiet" kalsın istiyorsanız bırakın; açılış sorunlarını görmek için boşaltın
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX=""

# Diğer işletim sistemlerini menüye ekle (tek OS ise true yapıp hızlandırın)
GRUB_DISABLE_OS_PROBER=false

# Snapshot girdilerinin "recovery" kopyaları menüyü şişirir
GRUB_DISABLE_RECOVERY=false

# grub-btrfs'in ürettiği alt menü için (Aşama 6)
GRUB_DISABLE_SUBMENU=false
EOF
```

`rootflags=subvol=@` **elle eklenmez**. GRUB'ın `10_linux` script'i kökün bir
BTRFS subvolume'ü üzerinde olduğunu tespit edip parametreyi kendisi ekler:

```
case x"$GRUB_FS" in
    xbtrfs)
        rootsubvol="`make_system_path_relative_to_its_root /`"
        ...
        GRUB_CMDLINE_LINUX="rootflags=subvol=${rootsubvol} ${GRUB_CMDLINE_LINUX}"
```

---

## 5.7 GRUB yapılandırmasının üretilmesi ve doğrulanması

```bash
update-grub          # = grub-mkconfig -o /boot/grub/grub.cfg
```

Doğrulayın:

```bash
# Her iki çekirdek de menüde mi?
grep -E "^\s*(menuentry|linux)\s" /boot/grub/grub.cfg | grep -E 'vmlinuz|menuentry' | head -20

# rootflags otomatik eklendi mi?
grep -o 'rootflags=subvol=[^ ]*' /boot/grub/grub.cfg | sort -u
# -> rootflags=subvol=@

# Kök UUID doğru mu?
grep -o "root=UUID=[a-f0-9-]*" /boot/grub/grub.cfg | sort -u
```

`rootflags=subvol=@` çıkmıyorsa **durun** — sistem açılmaz. Genellikle sebebi
chroot içinde `/` gerçekten `@` subvolume'ü üzerinde bağlı değildir:

```bash
findmnt -no SOURCE,FSTYPE /      # -> /dev/nvme1n1p2[/@]  btrfs
```

---

➡️ Sonraki: [Aşama 6 — Snapper, APT Hook, grub-btrfs, Bakım ve zram](06-snapper-bakim-zram.md)
