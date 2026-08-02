# Aşama 8 — Temizlik ve Kurulumun Tamamlanması

Hâlâ chroot içindeyiz. Buradaki sıralama önemlidir.

---

## 8.1 Son kontroller

```bash
echo "=== fstab ==="
cat /etc/fstab
findmnt --verify --verbose

echo "=== çekirdekler (2 adet olmalı) ==="
ls -1 /boot/vmlinuz-* /boot/initrd.img-*
dpkg -l | grep -E '^ii\s+linux-image-[0-9]' | awk '{print $2}'

echo "=== systemd bulaşması ==="
dpkg -l | grep -E '^ii\s+(systemd|systemd-sysv|libpam-systemd)\s' \
    && echo "!!! SORUN VAR !!!" || echo "temiz"

echo "=== init zinciri ==="
dpkg -l sysvinit-core openrc elogind eudev | grep '^ii'
grep -E '^(id|si|l2):' /etc/inittab
```

`findmnt --verify` fstab'daki her satırı denetler; hata verirse **yeniden
başlatmadan önce düzeltin**.

---

## 8.2 machine-id

D-Bus ve elogind kararlı bir makine kimliği ister:

```bash
if [ ! -s /etc/machine-id ]; then
    dbus-uuidgen --ensure=/etc/machine-id
fi
mkdir -p /var/lib/dbus
ln -sf /etc/machine-id /var/lib/dbus/machine-id

cat /etc/machine-id
```

---

## 8.3 OpenRC servis listesinin son senkronizasyonu

Kurulum boyunca eklenen paketlerin init script'lerinden `rc-update` ile
eklemeyi atladığınız olabilir. Aşama 4'te yazdığımız yardımcıyı çalıştırın:

```bash
openrc-sync-services
rc-update show
```

Listeyi gözden geçirin. Beklenmeyen bir şey varsa kaldırın:

```bash
# örnek: rc-update del <servis> <runlevel>
```

---

## 8.4 GRUB ve initramfs'in son kez üretilmesi

```bash
update-initramfs -u -k all
update-grub

# Doğrulama
grep -o 'rootflags=subvol=[^ ]*' /boot/grub/grub.cfg | sort -u   # -> rootflags=subvol=@
grep -c 'menuentry ' /boot/grub/grub.cfg
```

---

## 8.5 APT snapshot hook'unun açılması

Aşama 6.1'de geçici olarak kapatmıştık. **Şimdi açın** — chroot içindeki son
`apt` işlemlerini bitirdikten sonra:

```bash
sed -i 's/^DISABLE_APT_SNAPSHOT=.*/DISABLE_APT_SNAPSHOT="no"/' /etc/default/snapper
grep DISABLE_APT /etc/default/snapper      # -> "no"
```

Bundan sonra her `apt install/upgrade/remove` işlemi öncesinde `pre`, sonrasında
`post` anlık görüntüsü alınacak.

---

## 8.6 `policy-rc.d`'nin kaldırılması

> **Bu adımı atlamayın.** Dosya kalırsa kurulu sistem hiçbir servisi
> başlatamaz — ne LightDM, ne NetworkManager.

```bash
rm -f /usr/sbin/policy-rc.d
ls -l /usr/sbin/policy-rc.d 2>/dev/null && echo "!!! HÂLÂ DURUYOR !!!" || echo "silindi"
```

---

## 8.7 Paket önbelleğinin temizlenmesi

```bash
apt autoremove --purge -y
apt clean
rm -rf /var/lib/apt/lists/*

# Live ISO'dan kopyalanan DNS ayarını sıfırla (NetworkManager yeniden yazacak)
printf 'nameserver 1.1.1.1\nnameserver 9.9.9.9\n' > /etc/resolv.conf

du -sh /var/cache/apt
```

---

## 8.8 İlk referans anlık görüntüsü

Temiz kurulumun anlık görüntüsünü alın — bir şeyler bozulduğunda dönebileceğiniz
bilinen iyi durum:

```bash
snapper --no-dbus -c root create \
    --type single \
    --cleanup-algorithm number \
    --description "temiz kurulum - ilk acilis oncesi"

snapper --no-dbus -c root list
```

`--no-dbus` chroot'ta zorunludur (D-Bus çalışmıyor). Gerçek sistemde bayrak
gerekmez.

---

## 8.9 Chroot'tan çıkış ve unmount

```bash
exit          # chroot'tan çık — artık Arch Live ISO'dasınız
```

```bash
# Doğru yerde olduğunuzu teyit edin
cat /etc/os-release | head -2      # -> Arch Linux

# Bekleyen yazmaları diske indir
sync

# EFI değişkenleri bind-mount'u
umount -l /mnt/sys/firmware/efi/efivars 2>/dev/null || true

# Özyinelemeli unmount — --make-rslave sayesinde temiz çalışır
umount -R /mnt

# Hiçbir şey kalmadı mı?
findmnt -R /mnt 2>/dev/null || echo "/mnt temiz"
```

### `umount: target is busy` alırsanız

```bash
# Kim tutuyor?
fuser -vm /mnt
lsof +f -- /mnt 2>/dev/null | head

# Chroot içinde kalmış bir kabuk varsa kapatın; sonra:
umount -R -l /mnt      # lazy unmount — son çare
sync
```

BTRFS için son bir güvenlik adımı:

```bash
btrfs filesystem sync / 2>/dev/null || true
sync; sync
```

---

## 8.10 Yeniden başlatma

```bash
# ⚠️ Kurulum USB'sini şimdi çıkarın ya da UEFI önyükleme sırasını Devuan'a alın
systemctl reboot        # Arch Live ISO systemd kullanır — burada doğru komut budur
```

UEFI önyükleme girdisini kontrol etmek isterseniz (yeniden başlatmadan önce):

```bash
efibootmgr -v | grep -i devuan
# Girdiyi ilk sıraya almak için:
# efibootmgr -o <DevuanNum>,<digerleri>
```

---

## 8.11 İlk açılış kontrol listesi

Sisteme giriş yaptıktan sonra sırasıyla doğrulayın:

```bash
# 1) Init sistemi
ps -p 1 -o comm=              # -> init  (sysvinit)
rc-status                     # OpenRC servis tablosu
rc-status --all | head -40

# 2) Çalışan servisler
rc-service dbus status
rc-service elogind status
rc-service network-manager status
rc-service lightdm status

# 3) systemd yok
which systemctl || echo "systemctl YOK — doğru"
pidof systemd  || echo "systemd çalışmıyor — doğru"

# 4) Oturum yönetimi (elogind)
loginctl list-sessions
loginctl show-session "$(loginctl --no-legend list-sessions | awk 'NR==1{print $1}')" -p Type -p Active

# 5) BTRFS düzeni
findmnt -t btrfs
btrfs subvolume list /
btrfs filesystem usage /

# 6) TRIM (discard=async etkin mi)
findmnt -no OPTIONS /  | tr ',' '\n' | grep discard

# 7) zram
rc-service zramswap status
zramctl
swapon --show
cat /proc/sys/vm/swappiness       # -> 180

# 8) Ses
pactl info | head -5
pactl list short sinks

# 9) Snapper + APT hook
sudo snapper list
sudo snapper -c root get-config | head
grep DISABLE_APT /etc/default/snapper   # -> "no"

# 10) Çekirdekler
uname -r
ls -1 /boot/vmlinuz-*

# 11) Flatpak
flatpak remotes -d
```

### APT hook'unun canlı testi

```bash
sudo apt install --reinstall -y htop
sudo snapper list | tail -5
# "pre" ve "post" tipinde, açıklaması "apt" olan iki yeni satır görmelisiniz
```

### grub-btrfs testi

```bash
sudo snapper -c root create -d "grub-btrfs testi"
sleep 5
sudo rc-service grub-btrfsd status
sudo grep -c 'snapshots' /boot/grub/grub-btrfs.cfg
```

Yeniden başlattığınızda GRUB menüsünde **"Devuan Excalibur snapshots"** alt
menüsü görünmelidir.

---

## 8.12 Açılmazsa

| Belirti | Muhtemel sebep | Çözüm |
|---|---|---|
| GRUB menüsü hiç gelmiyor | UEFI girdisi yok / Secure Boot | UEFI ayarlarından Devuan girdisini seçin, Secure Boot'u kapatın |
| `Kernel panic - not syncing: VFS: Unable to mount root fs` | `rootflags=subvol=@` eksik | Live ISO'dan chroot'a girip `update-grub`; Aşama 5.7 |
| initramfs kabuğuna düşüyor (`(initramfs)`) | BTRFS modülü initrd'de yok | `update-initramfs -u -k all`; Aşama 5.4 |
| Açılıyor ama hiçbir servis başlamıyor | `policy-rc.d` silinmedi | `rm /usr/sbin/policy-rc.d`; Aşama 8.6 |
| Konsolda getty yok / tty kilitli | OpenRC `agetty` servisleri eklenmiş | `rc-update del agetty.tty1 default` …; Aşama 4.7 |
| Grafik oturum açılmıyor | LightDM runlevel'a eklenmedi | `rc-update add lightdm default` |
| `/home` boş | Home diski fstab'da yanlış UUID | `blkid` ile karşılaştırın; Aşama 5.1 |

Ayrıntılı kurtarma adımları:
[Ek — Rollback ve Sorun Giderme](09-ek-rollback-sorun-giderme.md)

---

✅ **Kurulum tamamlandı.**

Devuan 6.0 Excalibur; sysvinit + OpenRC, elogind, eudev, BTRFS snapshot
altyapısı, iki yedekli LTS çekirdek, zram, XFCE + LightDM, PulseAudio ve
Flatpak ile çalışır durumda.
