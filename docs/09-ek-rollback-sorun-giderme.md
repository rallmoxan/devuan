# Ek — Rollback, Bakım ve Sorun Giderme

---

## 1. Snapshot'a geri dönüş (rollback)

### 1.1 Neden `snapper rollback` bu düzende işe yaramaz

`snapper rollback`, BTRFS'in **varsayılan subvolume**'ünü (`btrfs subvolume
set-default`) değiştirerek çalışır. Bizim `fstab`'ımızda kök açıkça
`subvol=@` ile bağlanıyor ve GRUB `rootflags=subvol=@` geçiriyor — yani
varsayılan subvolume yok sayılır. Bu bilinçli bir tercihtir: `subvol=@` sabit
olduğu için sistem her koşulda öngörülebilir biçimde açılır.

Karşılığında rollback'i elle yaparız. Yöntem basit ve tersine çevrilebilir:
**bozuk `@`'yı yeniden adlandır, hedef snapshot'tan yeni bir `@` üret.**

### 1.2 Hangi snapshot'a döneceğinize karar verin

```bash
sudo snapper -c root list
```

```
 # | Tip    | Pre # | Tarih               | Kullanıcı | Temizlik | Açıklama
---+--------+-------+---------------------+-----------+----------+------------------
 0 | single |       |                     | root      |          | current
42 | pre    |       | 2026-08-02 14:10:03 | root      | number   | apt
43 | post   |    42 | 2026-08-02 14:11:47 | root      | number   | apt
```

Bir güncellemeden önceki hâle dönmek için **`pre`** numarasını seçin (örnekte 42).

### 1.3 GRUB'dan snapshot'a açılış (teşhis için)

Yeniden başlatın → **"Devuan Excalibur snapshots"** alt menüsü → istediğiniz
snapshot. Snapper snapshot'ları **salt okunurdur**; bu şekilde açılan sistem
teşhis için uygundur, kalıcı çalışma için değil. Buradan da rollback
yapabilirsiniz ama en temiz yol Live ISO'dur.

### 1.4 Kalıcı rollback (Arch Live ISO'dan)

```bash
# ⚠️ AYGIT ADINI KENDİ SİSTEMİNİZE GÖRE DEĞİŞTİRİN
ROOT_PART="/dev/nvme1n1p2"
SNAP_NUM=42                     # dönmek istediğiniz snapshot numarası

mkdir -p /mnt/btrfs-top
mount -o subvolid=5 "$ROOT_PART" /mnt/btrfs-top
ls /mnt/btrfs-top                       # -> @  @snapshots  @var_log

# Hedef snapshot gerçekten orada mı?
ls -d "/mnt/btrfs-top/@snapshots/$SNAP_NUM/snapshot"
cat "/mnt/btrfs-top/@snapshots/$SNAP_NUM/info.xml"
```

```bash
# 1) Mevcut kökü SİLME — yeniden adlandır (geri dönüş yolu açık kalsın)
mv "/mnt/btrfs-top/@" "/mnt/btrfs-top/@.bozuk-$(date +%Y%m%d-%H%M)"

# 2) Snapshot'tan YAZILABİLİR yeni kök üret (-r YOK => okuma/yazma)
btrfs subvolume snapshot \
    "/mnt/btrfs-top/@snapshots/$SNAP_NUM/snapshot" \
    "/mnt/btrfs-top/@"

# 3) Kontrol
btrfs subvolume list /mnt/btrfs-top
btrfs property get "/mnt/btrfs-top/@" ro      # -> ro=false

umount /mnt/btrfs-top
reboot
```

Açılış başarılıysa eski kökü silin:

```bash
sudo mkdir -p /mnt/btrfs-top
sudo mount -o subvolid=5 /dev/nvme1n1p2 /mnt/btrfs-top
sudo btrfs subvolume list /mnt/btrfs-top | grep bozuk
sudo btrfs subvolume delete /mnt/btrfs-top/@.bozuk-YYYYMMDD-HHMM
sudo umount /mnt/btrfs-top
```

**Açılmazsa** aynı adımlarla geri alın: yeni `@`'yı silip `@.bozuk-...`'yı
tekrar `@` yapın. Bu yüzden eski kökü hemen silmiyoruz.

### 1.5 Rollback sonrası

```bash
sudo update-grub                # menüyü yeni duruma göre yenile
sudo rc-service grub-btrfsd restart
sudo snapper -c root list
```

`/var/log` ve `/home` ayrı subvolume'lerde olduğu için **geriye sarmaz** —
rollback'in sebebini araştırırken günlükler ve kullanıcı verisi elinizde kalır.
Tasarımın amacı buydu.

---

## 2. Çekirdek bakımı — iki kernel'i yaşatmak

Devuan güncellemelerinde `linux-image-amd64` metapaketi yeni nokta sürümü
gösterir ve yeni bir `linux-image-6.12.<yeni>+deb13-amd64` kurulur. Eski
sürümler `apt autoremove` ile silinebilir.

Yedeği elde tutmak için:

```bash
# Kurulu çekirdekler
dpkg -l | grep -E '^ii\s+linux-image-[0-9]' | awk '{print $2}' | sort -V

# En yeni ikisini "manuel" işaretle -> autoremove dokunmaz
dpkg -l | grep -E '^ii\s+linux-image-[0-9]' | awk '{print $2}' | sort -V | tail -2 \
    | xargs -r sudo apt-mark manual

# Daha eskileri temizle (isterseniz)
dpkg -l | grep -E '^ii\s+linux-image-[0-9]' | awk '{print $2}' | sort -V | head -n -2 \
    | xargs -r echo "kaldırılabilir:"
```

Her çekirdek değişikliğinden sonra:

```bash
sudo update-initramfs -u -k all
sudo update-grub
ls -1 /boot/vmlinuz-*        # en az 2 tane kalmalı
```

`/boot` BTRFS `@` subvolume'ü içindedir; ayrı bir `/boot` bölümü olmadığı için
alan sıkıntısı yaşamazsınız.

---

## 3. Yeni bir servis paketi kurduğumda

> `update-rc.d` OpenRC runlevel'larını **güncellemez**. Yeni bir servis paketi
> her kurduğunuzda bunu yapın:

```bash
ls -l /etc/init.d/            # yeni gelen script'i görün
sudo rc-update add <servis> default
sudo rc-service <servis> start
rc-update show default
```

Toplu senkronizasyon için Aşama 4'te kurduğumuz yardımcı:

```bash
sudo openrc-sync-services
```

---

## 4. Sorun giderme

### 4.1 `apt` systemd kurmaya çalışıyor

```bash
apt-cache policy systemd systemd-sysv libpam-systemd
cat /etc/apt/preferences.d/no-systemd.pref
apt install -s <paket>          # simülasyon — ne yapacağını gösterir
```

Genellikle sebep bir Debian deposunun `sources.list`'e eklenmiş olmasıdır.
Devuan'ın merged deposu bu paketleri zaten içermez:

```bash
grep -rE '^deb ' /etc/apt/sources.list /etc/apt/sources.list.d/ | grep -v devuan
# Devuan dışı satır varsa kaldırın
```

### 4.2 Grafik oturum açılmıyor

```bash
sudo rc-service lightdm status
sudo tail -50 /var/log/lightdm/lightdm.log
sudo tail -50 /var/log/Xorg.0.log | grep -E '\(EE\)|\(WW\)'

# elogind oturumu görüyor mu?
loginctl list-sessions
sudo rc-service elogind restart

# X başlıyor mu? (kullanıcı olarak, konsoldan)
startxfce4
```

`elogind` çalışmıyorsa LightDM oturum açamaz — `Depends: default-logind |
logind | consolekit` bu yüzden vardır.

### 4.3 Ses yok

```bash
# Daemon çalışıyor mu?
pactl info
pgrep -a pulseaudio

# Elle başlat
start-pulseaudio-x11

# Autospawn init script'i eklenmiş mi?
rc-update show default | grep pulseaudio
cat /run/pulseaudio-enable-autospawn 2>/dev/null   # -> autospawn=yes

# Kart görünüyor mu / kısık mı?
aplay -l
alsamixer          # M tuşu ile susturmayı kaldırın

# Kullanıcı audio grubunda mı?
groups | tr ' ' '\n' | grep -x audio
```

### 4.4 Snapshot alınmıyor

```bash
# Yapılandırma var mı?
ls /etc/snapper/configs/
sudo snapper -c root get-config | grep -E 'TIMELINE_CREATE|NUMBER_CLEANUP'

# APT hook açık mı?
grep DISABLE_APT /etc/default/snapper       # -> "no" olmalı

# .snapshots doğru bağlı mı?
findmnt /.snapshots                          # -> [/@snapshots]

# snapperd D-Bus üzerinden başlıyor mu?
sudo rc-service dbus status
sudo snapper --no-dbus -c root list          # D-Bus'sız da çalışmalı

# cron işleri
ls -l /etc/cron.hourly/snapper-timeline /etc/cron.daily/snapper-cleanup
sudo run-parts --test /etc/cron.hourly
sudo rc-service cron status
```

### 4.5 GRUB'da snapshot menüsü yok

```bash
sudo rc-service grub-btrfsd status
sudo rc-service grub-btrfsd restart

# Daemon'un bağımlılığı
command -v inotifywait || sudo apt install -y inotify-tools

# Elle üret
sudo /etc/grub.d/41_snapshots-btrfs
sudo update-grub
ls -la /boot/grub/grub-btrfs.cfg

# Günlükler (--syslog ile çalışıyor)
sudo grep -i grub-btrfs /var/log/syslog | tail -20
```

Hiç snapshot yoksa alt menü de üretilmez — önce bir snapshot alın.

### 4.6 zram çalışmıyor

```bash
sudo rc-service zramswap status
sudo rc-service zramswap start
sudo rc-service zramswap stats

lsmod | grep zram
sudo modprobe zram
cat /sys/block/zram0/comp_algorithm     # köşeli parantez içindeki aktif olan

swapon --show
zramctl
```

`ALGO=zstd` ayarı çekirdeğiniz tarafından desteklenmiyorsa
`/etc/default/zramswap` içinde `ALGO=lz4` deneyin.

### 4.7 Disk doldu — snapshot'lar yer yiyor

```bash
btrfs filesystem usage /
sudo snapper -c root list

# Belirli aralığı sil
sudo snapper -c root delete 10-25

# Temizlik algoritmalarını hemen çalıştır
sudo snapper -c root cleanup number
sudo snapper -c root cleanup timeline
sudo snapper -c root cleanup empty-pre-post

# Limitleri sıkılaştır
sudo snapper -c root set-config NUMBER_LIMIT=10 TIMELINE_LIMIT_DAILY=3
```

> `df -h` BTRFS'te yanıltıcıdır (paylaşılan extent'ler nedeniyle). Her zaman
> `btrfs filesystem usage` kullanın.

### 4.8 Açılışta `fsck` / mount hataları

```bash
# Live ISO'dan
btrfs check --readonly /dev/nvme1n1p2

# Kurtarma mount'u (yalnızca veri kurtarmak için)
mount -o ro,rescue=usebackuproot /dev/nvme1n1p2 /mnt

# ⚠️ --repair SON ÇAREDİR ve veriyi bozabilir. Önce yedek alın.
```

---

## 5. Saf `openrc-init` PID 1 (isteğe bağlı)

Bu rehber `sysvinit-core`'u PID 1, OpenRC'yi servis yöneticisi olarak
kurgular — Devuan'ın desteklediği, en iyi test edilmiş bileşim budur.
OpenRC'nin kendi init'ini PID 1 yapmak isterseniz:

> ⚠️ Bunu **çalışan ve açıldığı doğrulanmış** bir sistemde yapın, kurulum
> sırasında değil. GRUB menüsünde eski girdi kalacağı için geri dönebilirsiniz.

```bash
# 1) getty'leri OpenRC'ye taşı (openrc-init /etc/inittab okumaz)
cd /etc/init.d
for t in tty1 tty2 tty3 tty4 tty5 tty6; do
    sudo ln -snf agetty "agetty.$t"
    sudo rc-update add "agetty.$t" default
done

# 2) inittab'daki getty satırlarını kapat (iki getty yarışmasın)
sudo sed -i -E 's|^([1-6]:[0-9]+:respawn:/sbin/getty)|#\1|' /etc/inittab
grep -n getty /etc/inittab
```

```bash
# 3) Çekirdek komut satırına ekle
sudo sed -i 's|^GRUB_CMDLINE_LINUX=.*|GRUB_CMDLINE_LINUX="init=/sbin/openrc-init"|' \
    /etc/default/grub
sudo update-grub
grep -o 'init=/sbin/openrc-init' /boot/grub/grub.cfg | head -1

sudo reboot
```

Açılıştan sonra:

```bash
ps -p 1 -o comm=          # -> openrc-init
rc-status
```

Kapatma/yeniden başlatma komutları değişir:

```bash
sudo openrc-shutdown --poweroff now
sudo openrc-shutdown --reboot now
```

**Geri dönmek için:** GRUB menüsünde `e` ile girdiyi düzenleyip
`init=/sbin/openrc-init` parametresini silin, açılın, `/etc/default/grub`
içindeki `GRUB_CMDLINE_LINUX`'u boşaltıp `update-grub` çalıştırın ve
`/etc/inittab`'daki getty satırlarının yorumunu kaldırın.

---

## 6. Faydalı komut özeti

| İş | systemd karşılığı | Bu sistemde |
|---|---|---|
| Servis başlat | `systemctl start x` | `rc-service x start` |
| Servis durdur | `systemctl stop x` | `rc-service x stop` |
| Durum | `systemctl status x` | `rc-service x status` |
| Açılışta etkinleştir | `systemctl enable x` | `rc-update add x default` |
| Açılıştan çıkar | `systemctl disable x` | `rc-update del x default` |
| Tüm servisler | `systemctl list-units` | `rc-status --all` |
| Runlevel içeriği | `systemctl get-default` | `rc-update show` |
| Günlükler | `journalctl -u x` | `grep x /var/log/syslog` |
| Açılış günlüğü | `journalctl -b` | `/var/log/rc.log` (`rc_logger="YES"`) |
| Kapat | `systemctl poweroff` | `shutdown -h now` / `poweroff` |
| Zamanlanmış iş | `systemd-timer` | `/etc/cron.{hourly,daily,weekly,monthly}` |
| Oturumlar | `loginctl` | `loginctl` (elogind aynı arayüzü sunar) |

---

⬅️ Geri: [Aşama 8 — Temizlik ve Kurulumun Tamamlanması](08-temizlik-reboot.md)
| 🏠 [Ana sayfa](../README.md)
