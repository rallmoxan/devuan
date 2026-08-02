# Aşama 6 — Snapper, APT Hook, grub-btrfs, Bakım Araçları ve zram

Bu aşama rehberin en kritik bölümüdür: burada kurulan mekanizmaların çoğu
systemd timer'ları yerine cron ve OpenRC servisleriyle karşılanır.

Chroot içinde devam ediyoruz.

---

## 6.1 Snapper kurulumu

```bash
apt install -y snapper
```

### APT anlık görüntülerini geçici olarak durdurun

`snapper` paketi APT hook'unu kendisi getirir (bkz. 6.3) ve bir sonraki adımda
`root` yapılandırması oluşur oluşmaz devreye girer. Chroot içinde D-Bus
çalışmadığı için bu hook her `apt` çağrısında boşa uğraşır. Kurulum bitene
kadar kapatıyoruz — **Aşama 8'de tekrar açılacak**:

```bash
sed -i 's/^DISABLE_APT_SNAPSHOT=.*/DISABLE_APT_SNAPSHOT="yes"/' /etc/default/snapper
grep DISABLE_APT /etc/default/snapper
```

---

## 6.2 `root` yapılandırması ve `@snapshots`'ın bağlanması

> **Sıralama önemlidir.** `snapper create-config`, `/.snapshots` **zaten varsa**
> (dizin ya da subvolume fark etmez) hata verip çıkar. Bu yüzden Aşama 2'de
> `@snapshots`'ı bağlamamış, Aşama 5'te fstab'a yazmamıştık.

```bash
# 1) Snapper kendi .snapshots subvolume'ünü oluştursun.
#    Chroot'ta D-Bus yok; --no-dbus şart.
snapper --no-dbus -c root create-config /

# "Cannot detect ambit since default subvolume is unknown" hatası alırsanız:
# snapper --no-dbus --ambit classic -c root create-config /

ls -la /etc/snapper/configs/root
btrfs subvolume list / | grep -i snapshots
```

```bash
# 2) Snapper'ın oluşturduğu subvolume'ü sil, yerine kendi @snapshots'ımızı koy
btrfs subvolume delete /.snapshots
mkdir -p /.snapshots

# 3) fstab'a ekle
UUID_ROOT=$(findmnt -no UUID -T /)
OPTS="noatime,compress=zstd:1,space_cache=v2,discard=async"

cat >> /etc/fstab <<EOF

# Snapshot deposu — @snapshots subvolume'ü
UUID=$UUID_ROOT   /.snapshots   btrfs   ${OPTS},subvol=@snapshots   0      0
EOF

# 4) Bağla ve izinleri sıkılaştır
mount /.snapshots
chmod 750 /.snapshots

findmnt /.snapshots
```

`findmnt` çıktısında kaynak `...[/@snapshots]` görünmelidir.

### Snapper `root` ayarları

```bash
snapper --no-dbus -c root set-config \
    TIMELINE_CREATE=yes \
    TIMELINE_CLEANUP=yes \
    TIMELINE_LIMIT_HOURLY=6 \
    TIMELINE_LIMIT_DAILY=7 \
    TIMELINE_LIMIT_WEEKLY=4 \
    TIMELINE_LIMIT_MONTHLY=3 \
    TIMELINE_LIMIT_YEARLY=0 \
    NUMBER_CLEANUP=yes \
    NUMBER_LIMIT=20 \
    NUMBER_LIMIT_IMPORTANT=10 \
    EMPTY_PRE_POST_CLEANUP=yes \
    SPACE_LIMIT=0.4 \
    FREE_LIMIT=0.2

grep -E '^(TIMELINE|NUMBER|SPACE|FREE)' /etc/snapper/configs/root
```

`NUMBER_LIMIT=20` → APT hook'unun ürettiği pre/post çiftlerinden en fazla 20
tanesi tutulur. `SPACE_LIMIT=0.4` → snapshot'lar dosya sisteminin %40'ını
aşarsa temizlik agresifleşir.

### `/home` için ayrı yapılandırma (isteğe bağlı)

`/home` ayrı bir diskte ve ayrı bir BTRFS dosya sistemi olduğu için kendi
snapshot yapılandırmasını alabilir:

```bash
snapper --no-dbus -c home create-config /home
snapper --no-dbus -c home set-config \
    TIMELINE_CREATE=yes TIMELINE_CLEANUP=yes \
    TIMELINE_LIMIT_HOURLY=4 TIMELINE_LIMIT_DAILY=7 \
    TIMELINE_LIMIT_WEEKLY=4 TIMELINE_LIMIT_MONTHLY=2 TIMELINE_LIMIT_YEARLY=0

# Her iki yapılandırmayı da zamanlayıcıya bildir
sed -i 's/^SNAPPER_CONFIGS=.*/SNAPPER_CONFIGS="root home"/' /etc/default/snapper
```

Yalnızca kök yapılandırmasını kullanacaksanız:

```bash
sed -i 's/^SNAPPER_CONFIGS=.*/SNAPPER_CONFIGS="root"/' /etc/default/snapper
```

> Bu `/home/.snapshots` subvolume'ünü home diskinin içinde oluşturur; kök
> snapshot'ından tamamen bağımsızdır ve rollback'ten etkilenmez.

---

## 6.3 APT hook — ek paket kurmaya gerek yok

Aranan `snapper-apt-plugin` / `80snapper` mekanizması **Devuan'ın `snapper`
paketiyle birlikte geliyor**. Doğrulayın:

```bash
dpkg -S /etc/apt/apt.conf.d/80snapper
# -> snapper: /etc/apt/apt.conf.d/80snapper

cat /etc/apt/apt.conf.d/80snapper
```

İçeriği (sadeleştirilmiş):

```
DPkg::Pre-Invoke  { "... snapper create -d apt -c number -t pre -p > /var/tmp/snapper-apt ;
                        snapper cleanup number ..." };
DPkg::Post-Invoke { "... snapper create -d apt -c number -t post
                        --pre-number=`cat /var/tmp/snapper-apt` ;
                        snapper cleanup number ..." };
```

Hook'un çalışması için gereken **üç koşul**:

1. `/usr/bin/snapper` mevcut — ✅ paketle geldi
2. `/etc/default/snapper` içinde `DISABLE_APT_SNAPSHOT` ≠ `"yes"` — Aşama 8'de açılacak
3. **`/etc/snapper/configs/root` mevcut** — ✅ 6.2'de oluşturuldu

Her komut `|| true` ile sarmalanmıştır: snapshot alınamazsa `apt` işlemi
kesilmez. Bu, kurulumun ortasında sistemi kilitlememek için bilinçli bir
tasarımdır.

---

## 6.4 Snapper zamanlayıcılarının cron karşılığı

`snapper` paketi **hiçbir init script'i getirmez**; yalnızca systemd unit ve
timer'ları vardır:

```
/usr/lib/systemd/system/snapper-timeline.timer   (OnCalendar=hourly)
/usr/lib/systemd/system/snapper-cleanup.timer    (günlük)
/usr/lib/systemd/system/snapper-boot.timer       (açılışta)
```

Bunların yaptığı işi cron ve OpenRC ile birebir yeniden kuruyoruz. Timer'ların
çağırdığı gerçek komutlar (unit dosyalarındaki `ExecStart` satırlarından):

| systemd birimi | Gerçek komut |
|---|---|
| `snapper-timeline.service` | `/usr/lib/snapper/systemd-helper --timeline` |
| `snapper-cleanup.service` | `/usr/lib/snapper/systemd-helper --cleanup` |
| `snapper-boot.service` | `snapper --config root create --cleanup-algorithm number --description "boot"` |

### Saatlik timeline

```bash
cat > /etc/cron.hourly/snapper-timeline <<'EOF'
#!/bin/sh
# systemd snapper-timeline.timer karşılığı (OnCalendar=hourly)
[ -x /usr/lib/snapper/systemd-helper ] || exit 0
exec /usr/lib/snapper/systemd-helper --timeline
EOF
chmod +x /etc/cron.hourly/snapper-timeline
```

### Günlük temizlik

```bash
cat > /etc/cron.daily/snapper-cleanup <<'EOF'
#!/bin/sh
# systemd snapper-cleanup.timer karşılığı
[ -x /usr/lib/snapper/systemd-helper ] || exit 0
exec /usr/lib/snapper/systemd-helper --cleanup
EOF
chmod +x /etc/cron.daily/snapper-cleanup
```

> `run-parts` nokta içeren dosya adlarını yok sayar. `snapper-timeline` gibi
> noktasız adlar kullanın; `snapper.timeline` çalışmaz.

### Açılış anlık görüntüsü — OpenRC servisi

```bash
cat > /etc/init.d/snapper-boot <<'EOF'
#!/sbin/openrc-run
# systemd snapper-boot.timer/service karşılığı

name="snapper boot snapshot"
description="Açılışta kök subvolume'ün anlık görüntüsünü alır"

depend() {
    need localmount
    after bootmisc dbus elogind
}

start() {
    ebegin "Açılış anlık görüntüsü alınıyor"
    /usr/bin/snapper --config root create \
        --cleanup-algorithm number \
        --description "boot"
    eend $?
}

stop() {
    return 0
}
EOF
chmod +x /etc/init.d/snapper-boot

rc-update add snapper-boot default
```

Doğrulama:

```bash
ls -l /etc/cron.hourly/snapper-timeline /etc/cron.daily/snapper-cleanup /etc/init.d/snapper-boot
rc-update show default | grep snapper
```

> `snapperd` D-Bus ile aktive olur
> (`/usr/share/dbus-1/system-services/org.opensuse.Snapper.service`), bu yüzden
> ayrıca bir daemon servisi eklemenize gerek yoktur — `dbus` servisi
> çalıştığı sürece kendiliğinden başlar.

---

## 6.5 grub-btrfs — kaynaktan, OpenRC modunda

> **`grub-btrfs` Debian/Devuan'da paketli DEĞİLDİR.** `excalibur`,
> `excalibur-backports`, `excalibur-proposed-updates` ve `ceres` süitlerinin
> hiçbirinde yoktur (doğrulandı). Kaynaktan derlenmelidir.
>
> İyi haber: üstkaynak Makefile **yerleşik OpenRC desteği** sunar.

```bash
apt install -y git make inotify-tools

mkdir -p /usr/local/src && cd /usr/local/src
git clone https://github.com/Antynea/grub-btrfs.git
cd grub-btrfs

# Sürüm sabitleyin (master'ı körlemesine kullanmayın)
git checkout v4.14
git log -1 --oneline
```

```bash
# ⚠️ Kritik: OPENRC=true SYSTEMD=false
make OPENRC=true SYSTEMD=false install
```

Kurulan dosyalar (Makefile'dan doğrulandı):

```
/etc/grub.d/41_snapshots-btrfs        # GRUB menü üreteci
/etc/default/grub-btrfs/config        # ayarlar
/usr/bin/grub-btrfsd                  # izleyici daemon
/etc/init.d/grub-btrfsd               # OpenRC init script'i
/etc/conf.d/grub-btrfsd               # OpenRC yapılandırması
/usr/share/man/man8/grub-btrfs.8
/usr/share/man/man8/grub-btrfsd.8
```

`SYSTEMD=false` verilmezse `/usr/lib/systemd/system/grub-btrfsd.service` de
kurulur — zararsız ama gereksizdir.

### Yapılandırma

```bash
# Alt menü adı
sed -i 's|^#\?GRUB_BTRFS_SUBMENUNAME=.*|GRUB_BTRFS_SUBMENUNAME="Devuan Excalibur snapshots"|' \
    /etc/default/grub-btrfs/config

# Menüde gösterilecek azami snapshot sayısı
sed -i 's|^#\?GRUB_BTRFS_LIMIT=.*|GRUB_BTRFS_LIMIT="25"|' /etc/default/grub-btrfs/config

grep -E '^GRUB_BTRFS_(SUBMENUNAME|LIMIT|IGNORE_SPECIFIC_PATH)' /etc/default/grub-btrfs/config
```

`GRUB_BTRFS_IGNORE_SPECIFIC_PATH=("@")` varsayılanı korunmalıdır — çalışan kök
subvolume'ünün snapshot listesine girmesini engeller.

### OpenRC daemon'u

`/etc/conf.d/grub-btrfsd` varsayılanı zaten doğru (snapper `/.snapshots`
kullanıyor):

```bash
cat /etc/conf.d/grub-btrfsd
# snapshots="/.snapshots"
# optional_args="--syslog"

rc-update add grub-btrfsd default
rc-update show default | grep grub-btrfsd
```

Daemon `inotifywait` ile `/.snapshots` dizinini izler; yeni bir snapshot
oluştuğunda GRUB menüsünü otomatik yeniler. `inotify-tools` kurulu değilse
şu hatayla çıkar: `[!] inotifywait was not found, exiting.`

### GRUB menüsünü üret

```bash
update-grub
grep -c "Devuan Excalibur snapshots" /boot/grub/grub.cfg || \
    echo "Not: henüz hiç snapshot yok — bu normaldir, ilk snapshot sonrası dolacak"
ls -la /boot/grub/grub-btrfs.cfg 2>/dev/null
```

---

## 6.6 btrfsmaintenance — cron modu

```bash
apt install -y btrfsmaintenance
```

Bu paket de yalnızca systemd timer'ları getirir, **ama** cron symlink'lerini
kuran bir script'i vardır: `/usr/share/btrfsmaintenance/btrfsmaintenance-refresh-cron.sh`.

```bash
# Günlükleri syslog'a yaz (journal yok)
sed -i 's|^BTRFS_LOG_OUTPUT=.*|BTRFS_LOG_OUTPUT="syslog"|' /etc/default/btrfsmaintenance

# Aylık scrub — her iki BTRFS diskini de kapsasın
sed -i 's|^BTRFS_SCRUB_PERIOD=.*|BTRFS_SCRUB_PERIOD="monthly"|'          /etc/default/btrfsmaintenance
sed -i 's|^BTRFS_SCRUB_MOUNTPOINTS=.*|BTRFS_SCRUB_MOUNTPOINTS="/:/home"|' /etc/default/btrfsmaintenance
sed -i 's|^BTRFS_SCRUB_PRIORITY=.*|BTRFS_SCRUB_PRIORITY="idle"|'          /etc/default/btrfsmaintenance

# Aylık balance
sed -i 's|^BTRFS_BALANCE_PERIOD=.*|BTRFS_BALANCE_PERIOD="monthly"|'            /etc/default/btrfsmaintenance
sed -i 's|^BTRFS_BALANCE_MOUNTPOINTS=.*|BTRFS_BALANCE_MOUNTPOINTS="/:/home"|'  /etc/default/btrfsmaintenance

# ⚠️ TRIM: discard=async ile çakışmasın diye KAPALI kalmalı
sed -i 's|^BTRFS_TRIM_PERIOD=.*|BTRFS_TRIM_PERIOD="none"|' /etc/default/btrfsmaintenance

# Defrag: CoW'lu snapshot düzeninde defrag paylaşılan extent'leri
# kopyalayarak disk kullanımını PATLATIR. Kapalı bırakın.
sed -i 's|^BTRFS_DEFRAG_PERIOD=.*|BTRFS_DEFRAG_PERIOD="none"|' /etc/default/btrfsmaintenance

# İşleri sıraya sok, aynı anda çalışmasınlar
sed -i 's|^BTRFS_ALLOW_CONCURRENCY=.*|BTRFS_ALLOW_CONCURRENCY="false"|' /etc/default/btrfsmaintenance
```

Cron symlink'lerini kur:

```bash
/usr/share/btrfsmaintenance/btrfsmaintenance-refresh-cron.sh cron
```

Doğrulama — script `/etc/cron.<period>/` altına symlink atar:

```bash
ls -l /etc/cron.monthly/btrfs-scrub /etc/cron.monthly/btrfs-balance
# -> /usr/share/btrfsmaintenance/btrfs-scrub.sh  vb.
ls -l /etc/cron.daily/btrfs-* /etc/cron.weekly/btrfs-* 2>/dev/null
```

> `/etc/default/btrfsmaintenance` dosyasını ileride değiştirirseniz
> `btrfsmaintenance-refresh-cron.sh cron` komutunu **tekrar çalıştırın**;
> symlink'ler kendiliğinden güncellenmez.

---

## 6.7 Grafik bakım araçları

```bash
apt install -y btrfs-assistant snapper-gui
```

| Araç | Not |
|---|---|
| `btrfs-assistant` 2.1.1 | Qt6 arayüz. `Depends: pkexec` → `polkitd` (Aşama 4'te kuruldu). Snapper yapılandırmalarını, subvolume'leri, balance/scrub işlerini yönetir |
| `snapper-gui` | GTK arayüz. Snapshot karşılaştırma ve geri alma |

Her ikisi de systemd'ye bağlı değildir; `snapperd`'a D-Bus üzerinden bağlanır.
Grafik oturumda `dbus` ve `elogind` servisleri çalıştığı sürece sorunsuz
çalışırlar.

---

## 6.8 zram — OpenRC init script'i

`zram-tools` paketi `/usr/sbin/zramswap` (start/stop/status alan bir script) ve
`/etc/default/zramswap` getirir, **ama yalnızca bir systemd unit'i** içerir —
init script'i yoktur. Kendimiz yazıyoruz.

```bash
apt install -y zram-tools
```

### Ayarlar

```bash
cat > /etc/default/zramswap <<'EOF'
# Sıkıştırma algoritması: zstd daha iyi sıkıştırır, lz4 daha hızlıdır.
# Modern CPU'larda zstd'nin ek yükü ihmal edilebilir.
ALGO=zstd

# Toplam RAM'in yüzde kaçı zram'e ayrılsın (SIZE'ı geçersiz kılar)
PERCENT=50

# PERCENT tanımlıysa kullanılmaz; MiB cinsinden sabit boyut
SIZE=512

# Disk üzerindeki swap'tan yüksek olmalı ki önce zram kullanılsın
PRIORITY=100
EOF
```

### Init script

```bash
cat > /etc/init.d/zramswap <<'EOF'
#!/sbin/openrc-run
# zram-tools için OpenRC servisi.
# Paketin getirdiği /usr/lib/systemd/system/zramswap.service karşılığı.

name="zram swap"
description="zram üzerinde sıkıştırılmış takas alanı"
extra_commands="stats"

depend() {
    need localmount
    after modules kmod
    before bootmisc
}

start() {
    ebegin "zram takas alanı etkinleştiriliyor"
    /usr/sbin/zramswap start
    eend $?
}

stop() {
    ebegin "zram takas alanı kapatılıyor"
    /usr/sbin/zramswap stop
    eend $?
}

stats() {
    /usr/sbin/zramswap status
}
EOF
chmod +x /etc/init.d/zramswap

rc-update add zramswap boot
rc-update show boot | grep zramswap
```

`default` yerine `boot` runlevel'ı seçildi: takas alanı, kullanıcı servisleri
başlamadan önce hazır olsun.

### Çekirdek modülü

`zramswap start` komutu `modprobe zram` çalıştırır. Modülü açılışta hazır
bulundurmak için:

```bash
echo "zram" > /etc/modules-load.d/zram.conf
```

### Sanal bellek ayarı

zram'in gerçekten kullanılabilmesi için çekirdeğe takas alanını daha erken
kullanmasını söyleyin — RAM'e göre hızlı olduğu için bu doğru davranıştır:

```bash
cat > /etc/sysctl.d/99-zram.conf <<'EOF'
# zram, disk swap'ından çok daha hızlıdır; agresif swap doğru seçimdir
vm.swappiness = 180
vm.watermark_boost_factor = 0
vm.watermark_scale_factor = 125
vm.page-cluster = 0
EOF
```

> `vm.swappiness=180`, çekirdek 5.8+ ile birlikte üst sınırın 200'e çıkması
> sayesinde geçerlidir (6.12 LTS kullanıyoruz). Disk swap'ı olan sistemlerde
> bu değeri 60–100 aralığında tutun.

Yeniden başlatmadan sonra kontrol:

```bash
# (kurulum bittikten sonra, gerçek sistemde)
# rc-service zramswap status
# rc-service zramswap stats
# zramctl
# swapon --show
```

---

## 6.9 Aşama 6 kontrol listesi

```bash
echo "--- snapper ---"
ls /etc/snapper/configs/
findmnt /.snapshots
grep DISABLE_APT /etc/default/snapper          # şu an "yes" (Aşama 8'de "no")

echo "--- cron işleri ---"
ls -l /etc/cron.hourly/snapper-timeline /etc/cron.daily/snapper-cleanup
ls -l /etc/cron.monthly/btrfs-* 2>/dev/null

echo "--- OpenRC servisleri ---"
rc-update show | grep -E 'snapper-boot|grub-btrfsd|zramswap'

echo "--- grub-btrfs ---"
ls -l /etc/grub.d/41_snapshots-btrfs /usr/bin/grub-btrfsd /etc/init.d/grub-btrfsd
```

---

➡️ Sonraki: [Aşama 7 — XFCE, PulseAudio, Flatpak ve Kullanıcı Ayarları](07-xfce-ses-flatpak.md)
