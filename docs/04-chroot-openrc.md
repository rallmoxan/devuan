# Aşama 4 — Chroot Ortamına Geçiş ve Temel OpenRC Yapılandırması

---

## 4.1 Sanal dosya sistemlerinin bağlanması

`--make-rslave`, chroot içindeki mount işlemlerinin Live ISO'nun mount
ağacına sızmasını engeller — Aşama 8'deki unmount'u sorunsuz kılan şey budur.

```bash
source /root/devuan-env.sh

mount --types proc  /proc            /mnt/proc
mount --rbind       /sys             /mnt/sys   && mount --make-rslave /mnt/sys
mount --rbind       /dev             /mnt/dev   && mount --make-rslave /mnt/dev
mount --rbind       /run             /mnt/run   && mount --make-rslave /mnt/run

# EFI değişkenleri (grub-install ve efibootmgr için).
# --rbind /sys bunu genellikle zaten kapsar; kapsamadıysa emniyet olsun diye:
mountpoint -q /mnt/sys/firmware/efi/efivars \
    || mount --bind /sys/firmware/efi/efivars /mnt/sys/firmware/efi/efivars 2>/dev/null \
    || true

# DNS
cp --dereference /etc/resolv.conf /mnt/etc/resolv.conf
```

Chroot'a girin:

```bash
chroot /mnt /usr/bin/env -i \
    HOME=/root TERM="$TERM" \
    PATH=/usr/sbin:/usr/bin:/sbin:/bin \
    LC_ALL=C.UTF-8 \
    /bin/bash --login
```

> **Kısayol:** `arch-install-scripts` kuruluysa yukarıdaki mount bloğunun ve
> chroot komutunun tamamı yerine `arch-chroot /mnt` yeterlidir; bind-mount'ları
> ve `resolv.conf`'u kendisi halleder.

Bundan sonraki tüm komutlar `(chroot)` içinde çalışır. Kontrol:

```bash
cat /etc/devuan_version        # -> excalibur
```

---

## 4.2 Devuan `sources.list`

```bash
cat > /etc/apt/sources.list <<'EOF'
# Devuan 6.0 "Excalibur" — birleşik (merged) depo
deb http://deb.devuan.org/merged excalibur          main contrib non-free non-free-firmware
deb http://deb.devuan.org/merged excalibur-updates  main contrib non-free non-free-firmware
deb http://deb.devuan.org/merged excalibur-security main contrib non-free non-free-firmware

# Kaynak paketler (grub-btrfs derlerken gerekmez; isteğe bağlı)
# deb-src http://deb.devuan.org/merged excalibur main contrib non-free non-free-firmware
EOF

cat /etc/apt/sources.list
```

> **`deb.devuan.org` vs `pkgmaster.devuan.org`:** `deb.devuan.org` round-robin
> ayna havuzudur ve günlük kullanım için doğru adrestir. `pkgmaster` ana
> sunucudur; onu yalnızca Aşama 1'de aynalama gecikmesi yaşamamak için
> kullandık.
>
> **Devuan'da güvenlik süiti `excalibur-security`dir** — Debian'daki
> `trixie-security`/`bookworm-security` biçimi burada geçerli değildir.

`non-free-firmware` bileşeninin gerçekten var olduğu doğrulanmıştır
(`dists/excalibur-security/` altında `main`, `contrib`, `non-free`,
`non-free-firmware`).

---

## 4.3 systemd'nin kalıcı olarak yasaklanması

> **Not:** Devuan merged deposunda `systemd`, `systemd-sysv` ve `libpam-systemd`
> paketleri **zaten yoktur** (2026-08-02 itibarıyla doğrulandı — amprolla
> bunları ayıklıyor). Aşağıdaki pin, ileride yanlışlıkla bir Debian deposu
> eklenirse devreye girecek emniyet kemeridir.

```bash
cat > /etc/apt/preferences.d/no-systemd.pref <<'EOF'
# Devuan/OpenRC sistemine systemd'nin hiçbir koşulda girmemesi için.
# -1 önceliği paketi kurulamaz yapar (bir bağımlılık zorlasa bile apt reddeder).

Package: systemd systemd-sysv systemd-boot systemd-container systemd-timesyncd systemd-resolved systemd-cron systemd-homed systemd-oomd
Pin: release *
Pin-Priority: -1

Package: libpam-systemd
Pin: release *
Pin-Priority: -1

# elogind ve eudev'in her zaman tercih edilmesi
Package: elogind libelogind0 libpam-elogind eudev
Pin: release o=Devuan
Pin-Priority: 1001
EOF
```

### ⚠️ Bu paketleri YASAKLAMAYIN

| Paket | Neden dokunulmamalı |
|---|---|
| `libsystemd0` | Salt bir kütüphane shim'i. `pulseaudio`, `rtkit`, `util-linux` ve düzinelerce paket buna bağlı. Yasaklamak sistemi kurulamaz hale getirir |
| `systemd-standalone-sysusers` | `Conflicts: systemd`. Bazı paketlerin `systemd | systemd-sysusers` alternatifini systemd'siz karşılayan paket |
| `udev` | Devuan'da bu, `eudev`'e bağlı bir geçiş paketidir |

Pin'i doğrulayın:

```bash
apt update
apt-cache policy systemd systemd-sysv libpam-systemd libsystemd0
# systemd*: "candidate: (none)" veya priority -1 görmelisiniz
# libsystemd0: normal öncelik (500) — böyle olmalı
```

---

## 4.4 Chroot içinde servislerin başlatılmasının engellenmesi

`apt` paket kurarken init script'lerini başlatmayı dener; chroot'ta bu ya
başarısız olur ya da ana sistemi bozar.

```bash
cat > /usr/sbin/policy-rc.d <<'EOF'
#!/bin/sh
exit 101
EOF
chmod +x /usr/sbin/policy-rc.d
```

> Bu dosya **Aşama 8'de silinecektir**. Silmeyi unutursanız kurulu sistem hiçbir
> servisi başlatamaz.

---

## 4.5 Yerelleştirme, saat dilimi, makine adı

```bash
apt update
apt install -y locales tzdata console-setup keyboard-configuration

# Yerel ayarlar
sed -i 's/^# *\(en_US.UTF-8\)/\1/; s/^# *\(tr_TR.UTF-8\)/\1/' /etc/locale.gen
locale-gen
cat > /etc/default/locale <<'EOF'
LANG=tr_TR.UTF-8
LC_MESSAGES=en_US.UTF-8
EOF

# Saat dilimi
ln -sf /usr/share/zoneinfo/Europe/Istanbul /etc/localtime
echo "Europe/Istanbul" > /etc/timezone
dpkg-reconfigure -f noninteractive tzdata

# Konsol klavyesi (Türkçe Q)
cat > /etc/default/keyboard <<'EOF'
XKBMODEL="pc105"
XKBLAYOUT="tr"
XKBVARIANT=""
XKBOPTIONS=""
BACKSPACE="guess"
EOF

# ⚠️ Makine adını kendinize göre değiştirin
echo "devuan-excalibur" > /etc/hostname
cat > /etc/hosts <<'EOF'
127.0.0.1   localhost
127.0.1.1   devuan-excalibur
::1         localhost ip6-localhost ip6-loopback
ff02::1     ip6-allnodes
ff02::2     ip6-allrouters
EOF
```

Donanım saati UTC (Linux-only makinelerde doğru seçim):

```bash
cat > /etc/adjtime <<'EOF'
0.0 0 0.0
0
UTC
EOF
```

---

## 4.6 OpenRC + elogind + eudev kurulumu

```bash
apt install -y \
    openrc \
    sysvinit-core \
    elogind libpam-elogind \
    eudev \
    polkitd \
    dbus dbus-x11 \
    rsyslog cron anacron logrotate \
    inotify-tools \
    btrfs-progs \
    ca-certificates gnupg \
    nano less bash-completion
```

Kurulum sırasında görecekleriniz ve anlamları:

- **`Removing sysv-rc ...`** — `openrc` paketi `Conflicts: sysv-rc` /
  `Replaces: sysv-rc`. Beklenen davranış. OpenRC artık runlevel mekanizmasıdır.
- **`Add existing services ...`** — `openrc` postinst'i mevcut `/etc/rc?.d/S*`
  bağlarını OpenRC runlevel'larına taşır. Eşleme: `rc1.d`→`recovery`,
  `rc2.d`→`default`, `rcS.d`→`sysinit`.
- **`WARNING: if you are replacing sysv-rc by OpenRC ... reboot immediately`** —
  bu uyarı *çalışan* sistemler içindir. Chroot'ta güvenle yok sayılır.

### Mimari: kim ne yapıyor?

```
sysvinit-core  →  PID 1. /etc/inittab'ı okur, getty'leri respawn eder.
                  inittab satırı:  l2:2:wait:/etc/init.d/rc 2
openrc         →  /etc/init.d/rc, "openrc default" çağıran ince bir sarmalayıcı.
                  Servis bağımlılıklarını çözer, /etc/runlevels/* dizinlerini okur.
elogind        →  Oturum/seat yönetimi, PolicyKit entegrasyonu, güç tuşları.
eudev          →  Aygıt düğümleri ve udev kuralları (udev'i Provides eder).
```

Kontrol:

```bash
dpkg -l openrc sysvinit-core elogind libpam-elogind eudev | grep '^ii'
ls /etc/runlevels/          # boot default nonetwork off recovery shutdown sysinit
test -f /etc/inittab && grep -E '^(id|si|l2):' /etc/inittab
```

---

## 4.7 Temel servislerin runlevel'lara eklenmesi

```bash
rc-update add elogind        default
rc-update add dbus           default
rc-update add eudev          sysinit
rc-update add cron           default
rc-update add rsyslog        default

rc-update show
```

### 🔴 Bilinmesi zorunlu: `update-rc.d` OpenRC'yi güncellemez

Debian'ın `openrc` paketi `update-rc.d`/`invoke-rc.d`'yi **saptırmaz**
(0.20.4-1'deki dpkg-divert'ler postinst tarafından kaldırılıyor). Sonuç:

> Bir paket kurduğunuzda init script'i `/etc/init.d/` altına gelir ve
> `update-rc.d` `/etc/rc?.d/` symlink'lerini oluşturur — **ama OpenRC bu
> dizinleri okumaz.** Servis, siz `rc-update add` demedikçe açılışta başlamaz.

Bu yüzden rehberde her paket kurulumundan sonra açıkça `rc-update add` çağrısı
yapılır. Bir şeyi atlamadığınızdan emin olmak için, `openrc` postinst'inin
mantığını tekrarlayan şu yardımcıyı kullanabilirsiniz:

```bash
cat > /usr/local/sbin/openrc-sync-services <<'EOF'
#!/bin/sh
# /etc/rc{S,1,2}.d altındaki sysv-rc bağlarını OpenRC runlevel'larına taşır.
# Debian openrc postinst'indeki mantığın aynısı.
set -e
for rl in S 1 2; do
    [ -d "/etc/rc${rl}.d" ] || continue
    case "$rl" in
        S) orl=sysinit ;;
        1) orl=recovery ;;
        2) orl=default ;;
    esac
    for f in /etc/rc${rl}.d/S*; do
        [ -e "$f" ] || continue
        svc=$(basename "$(readlink -f "$f")")
        [ -x "/etc/init.d/$svc" ] || continue
        # "[ -e ... ] && continue" YAZMAYIN: test başarısız olduğunda liste 1
        # döndürür ve "set -e" script'i sonlandırır.
        if [ -e "/etc/runlevels/$orl/$svc" ]; then
            continue
        fi
        echo "  + $svc -> $orl"
        rc-update add "$svc" "$orl" >/dev/null 2>&1 || true
    done
done
rc-update -u
EOF
chmod +x /usr/local/sbin/openrc-sync-services

openrc-sync-services
```

Kurulum bittikten sonra ve ileride yeni bir servis paketi kurduğunuzda bu
komutu çalıştırıp `rc-update show` ile sonucu gözden geçirin.

### getty'ler: OpenRC'nin `agetty` servisini **eklemeyin**

PID 1 `sysvinit` olduğu için konsol getty'leri `/etc/inittab` tarafından
respawn edilir:

```
1:2345:respawn:/sbin/getty --noclear 38400 tty1
2:23:respawn:/sbin/getty 38400 tty2
...
```

Buna ek olarak `rc-update add agetty.tty1 default` yaparsanız aynı tty'de iki
getty yarışır ve konsol kullanılamaz hale gelir. OpenRC'nin `agetty` servisleri
yalnızca `openrc-init` PID 1 olduğunda gerekir (bkz.
[Ek — 5. Saf `openrc-init` PID 1](09-ek-rollback-sorun-giderme.md#5-saf-openrc-init-pid-1-isteğe-bağlı)).

---

## 4.8 OpenRC paralel açılışı ve `lsb-base` düzeltmesi

`openrc` paketinin `README.Debian` dosyasından:

> `lsb-base: /lib/lsb/init-functions.d/20-left-info-blocks` messes up the output
> of OpenRC parallel boot. To disable this fancy output, move the file out of
> `/lib/lsb/init-functions.d`.

```bash
if [ -f /lib/lsb/init-functions.d/20-left-info-blocks ]; then
    mv /lib/lsb/init-functions.d/20-left-info-blocks /root/20-left-info-blocks.disabled
fi
```

Paralel açılışı etkinleştirmek isterseniz (isteğe bağlı, açılışı hızlandırır):

```bash
sed -i 's/^#\?rc_parallel=.*/rc_parallel="YES"/' /etc/rc.conf
grep -E '^rc_(parallel|logger)' /etc/rc.conf
```

Açılış günlüğü tutmak için:

```bash
sed -i 's/^#\?rc_logger=.*/rc_logger="YES"/' /etc/rc.conf
# -> /var/log/rc.log
```

---

➡️ Sonraki: [Aşama 5 — Çekirdek, Bootloader ve fstab](05-kernel-grub-fstab.md)
