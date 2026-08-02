# Aşama 7 — XFCE, LightDM, PulseAudio, Flatpak ve Kullanıcı Ayarları

Chroot içinde devam ediyoruz.

---

## 7.1 Xorg ve XFCE

```bash
apt install -y \
    xorg xserver-xorg xinit \
    xfce4 xfce4-goodies xfce4-terminal \
    xfce4-power-manager xfce4-notifyd \
    thunar-volman gvfs gvfs-backends udisks2 \
    xdg-user-dirs xdg-user-dirs-gtk \
    mate-polkit \
    desktop-base tango-icon-theme \
    fonts-dejavu fonts-liberation2 fonts-noto-core
```

`xfce4` metapaketinin çektikleri (doğrulanmış bağımlılıklar): `thunar`,
`xfce4-panel`, `xfce4-session`, `xfce4-settings`, `xfconf`, `xfdesktop4`,
`xfwm4`, `xfce4-appfinder`, `libxfce4ui-utils`, `xfce4-pulseaudio-plugin`.

---

## 7.2 Wayland'in devre dışı bırakılması

XFCE 4.20'de Wayland desteği **deneyseldir ve varsayılan değildir**; `xfce4`
metapaketi bir Wayland bileşimcisi (compositor) çekmez. Yine de kesinleştirelim.

### Kontrol: Wayland oturumu kurulmuş mu?

```bash
ls -la /usr/share/wayland-sessions/ 2>/dev/null || echo "Wayland oturumu YOK — istenen durum"
ls -la /usr/share/xsessions/
# -> xfce.desktop  (yalnızca X11 oturumu görünmeli)

# Bir bileşimci sızmış mı?
dpkg -l | grep -E '^ii\s+(labwc|wayfire|sway|weston|mutter|kwin-wayland)\s' \
    || echo "Wayland bileşimcisi YOK — istenen durum"
```

> `libgtk-layer-shell0` paketinin kurulu olması normaldir: `xfce4-session`
> bağımlılığıdır ve yalnızca bir kütüphanedir, oturumu Wayland'e çevirmez.

### Araçları X11'e sabitleyin

```bash
cat > /etc/profile.d/99-force-x11.sh <<'EOF'
# Bu sistem X11-only'dir. Araç setlerinin Wayland arka ucunu denemesini engelle.
export GDK_BACKEND=x11
export QT_QPA_PLATFORM=xcb
export CLUTTER_BACKEND=x11
export SDL_VIDEODRIVER=x11
export MOZ_ENABLE_WAYLAND=0
export XDG_SESSION_TYPE=x11
EOF
chmod +x /etc/profile.d/99-force-x11.sh
```

---

## 7.3 LightDM

```bash
apt install -y lightdm lightdm-gtk-greeter lightdm-gtk-greeter-settings
```

`lightdm` paketi `Depends: default-logind | logind | consolekit` der;
`libpam-elogind` `default-logind`'i **Provides** ettiği için elogind bu koşulu
karşılar (doğrulandı).

```bash
mkdir -p /etc/lightdm/lightdm.conf.d

cat > /etc/lightdm/lightdm.conf.d/50-devuan.conf <<'EOF'
[Seat:*]
greeter-session=lightdm-gtk-greeter
user-session=xfce
session-wrapper=/etc/X11/Xsession
allow-guest=false
EOF

# Devuan'ın init script'i var mı? (doğrulandı: evet)
ls -l /etc/init.d/lightdm

rc-update add lightdm default
```

`x-session-manager` alternatifinin XFCE'yi gösterdiğini teyit edin:

```bash
update-alternatives --list x-session-manager
update-alternatives --set x-session-manager /usr/bin/xfce4-session
```

---

## 7.4 Ağ — NetworkManager

```bash
apt install -y network-manager network-manager-gnome
```

`ifupdown` ile çakışmayı önleyin: `/etc/network/interfaces` içinde `lo`
dışındaki arayüzler tanımlıysa NetworkManager onları yönetmez.

```bash
cat > /etc/network/interfaces <<'EOF'
# Fiziksel arayüzler NetworkManager tarafından yönetilir.
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback
EOF

# NetworkManager yönetilmeyen aygıt bırakmasın
cat > /etc/NetworkManager/conf.d/10-globally-managed-devices.conf <<'EOF'
[keyfile]
unmanaged-devices=none
EOF

rc-update add network-manager default
```

Kablosuz için gerekli olabilecek firmware (donanımınıza göre):

```bash
# apt install -y firmware-iwlwifi        # Intel
# apt install -y firmware-realtek        # Realtek
# apt install -y firmware-atheros        # Atheros/QCA
```

---

## 7.5 Ses — PulseAudio (PipeWire değil)

```bash
apt install -y \
    pulseaudio pulseaudio-utils pulseaudio-module-bluetooth \
    pavucontrol alsa-utils rtkit
```

### PipeWire'ın gelmediğini doğrulayın

```bash
dpkg -l | grep -E '^ii\s+(pipewire|pipewire-pulse|wireplumber)\s' \
    || echo "PipeWire YOK — istenen durum"

# Ses sunucusunu kim sağlıyor?
dpkg -l pulseaudio | grep '^ii'
```

`xfce4-pulseaudio-plugin` yalnızca `Recommends: pavucontrol, pulseaudio |
pipewire-pulse` der. İlk alternatif PulseAudio olduğu ve onu açıkça kurduğumuz
için PipeWire hiçbir zaman devreye girmez. Bir daha girmesin diye pin koyalım:

```bash
cat > /etc/apt/preferences.d/no-pipewire.pref <<'EOF'
# Ses mimarisi PulseAudio'dur. PipeWire'ın ses sunucusu rolünü devralmasını engelle.
Package: pipewire-pulse pipewire-alsa wireplumber pipewire-media-session
Pin: release *
Pin-Priority: -1
EOF
```

> `pipewire` **kütüphanesini** yasaklamayın: bazı uygulamalar (ör. ekran
> paylaşımı yapan tarayıcılar) `libpipewire-0.3-0`'a bağlıdır. Yalnızca ses
> sunucusu rolünü üstlenen paketler engellenmiştir.

### Autospawn — OpenRC servisi

PulseAudio kullanıcı oturumu başına çalışır. Devuan'ın `pulseaudio` paketi bunun
için **bir init script'i getirir** (doğrulandı):

```bash
cat /etc/init.d/pulseaudio-enable-autospawn
# start) echo "autospawn=yes" > /run/pulseaudio-enable-autospawn

rc-update add pulseaudio-enable-autospawn default
```

Oturum açıldığında `/etc/xdg/autostart/pulseaudio.desktop`
(`Exec=start-pulseaudio-x11`) daemon'u başlatır. Yani systemd user session'a
ihtiyaç yoktur.

### Gerçek zamanlı öncelik

`rtkit`, PulseAudio'ya RT önceliği vermek için PolicyKit kullanır ve
`polkitd` + D-Bus üzerinden çalışır — systemd gerektirmez. Ek olarak PAM
limitleri:

```bash
cat > /etc/security/limits.d/95-pulseaudio.conf <<'EOF'
@audio   -  rtprio      95
@audio   -  memlock     unlimited
@audio   -  nice        -19
EOF
```

Bluetooth ses kullanacaksanız:

```bash
apt install -y bluez blueman
rc-update add bluetooth default
```

---

## 7.6 Flatpak ve Flathub

```bash
apt install -y flatpak xdg-desktop-portal xdg-desktop-portal-gtk

flatpak remote-add --if-not-exists flathub \
    https://dl.flathub.org/repo/flathub.flatpakrepo

flatpak remotes -d
```

> Chroot içinde `flatpak remote-add` ağ ya da D-Bus nedeniyle başarısız olursa
> sorun değil — **ilk açılıştan sonra** aynı komutu çalıştırın. Depo tanımı
> `/var/lib/flatpak/repo/config` içinde saklanır.

XFCE'nin Flatpak uygulamalarını menüde göstermesi için:

```bash
cat > /etc/profile.d/flatpak-xdg-dirs.sh <<'EOF'
export XDG_DATA_DIRS="${XDG_DATA_DIRS:-/usr/local/share:/usr/share}:/var/lib/flatpak/exports/share"
EOF
chmod +x /etc/profile.d/flatpak-xdg-dirs.sh
```

Portal'ların X11 oturumunda çalışması için `xdg-desktop-portal-gtk` yeterlidir;
`xdg-desktop-portal-wlr` **kurmayın**.

---

## 7.7 Kullanıcı hesapları

### Sudo yetkili kullanıcı

```bash
apt install -y sudo

# ⚠️ KULLANICI ADINI KENDİNİZE GÖRE DEĞİŞTİRİN
USERNAME="kullanici"

adduser --gecos "" "$USERNAME"          # parola sorulacak

usermod -aG sudo,audio,video,plugdev,netdev,cdrom,dip,scanner,lp "$USERNAME"
id "$USERNAME"
```

`sudo` grubunun gerçekten yetkili olduğunu doğrulayın:

```bash
grep -E '^%sudo' /etc/sudoers
# -> %sudo   ALL=(ALL:ALL) ALL
```

Satır yoksa (bir `sudoers` düzenlemesi bozulmuşsa) `visudo` ile ekleyin —
`/etc/sudoers`'ı asla doğrudan bir editörle açmayın.

### Snapper'a kullanıcı erişimi

```bash
snapper --no-dbus -c root set-config ALLOW_USERS="$USERNAME" SYNC_ACL=yes
snapper --no-dbus -c home set-config ALLOW_USERS="$USERNAME" SYNC_ACL=yes 2>/dev/null || true

grep -E '^(ALLOW_USERS|SYNC_ACL)' /etc/snapper/configs/root
```

`SYNC_ACL=yes`, `/.snapshots` üzerindeki ACL'leri yapılandırmaya göre günceller;
kullanıcı `sudo` olmadan da snapshot listeleyebilir.

### 🔒 Root hesabının kilitlenmesi

> **Bu adımı en sona bırakın.** Sudo yetkili kullanıcının çalıştığını
> doğrulamadan root'u kilitlerseniz sisteme yönetici olarak giremezsiniz.

Önce sudo'nun gerçekten çalıştığını doğrulayın:

```bash
su - "$USERNAME" -c 'sudo -n true 2>&1 | head -1; echo "sudo kullanılabilir"'
# Parola sorulması normaldir; "user is not in the sudoers file" görürseniz DURUN.
```

Sonra root'u kilitleyin:

```bash
# İsteğe bağlı: tek kullanıcı modu için bir root parolası belirlemek isterseniz
# önce şunu çalıştırın — kilitleme parolayı geçersiz kılmaz, yalnızca devre dışı
# bırakır ve "passwd -u root" ile geri açılabilir.
# passwd root

passwd -l root

passwd -S root
# -> root L ...      "L" = locked
```

Kurtarma erişimi hâlâ mümkündür: `sysvinit`'in `/etc/inittab` dosyasındaki

```
~~:S:wait:/sbin/sulogin --force
```

satırındaki `--force`, root hesabı kilitliyken bile tek kullanıcı moduna girişe
izin verir. Bu **bilinçli bir tasarım**dır: makineye fiziksel erişimi olan biri
GRUB üzerinden `single` parametresiyle root kabuğu alabilir. Bunu istemiyorsanız
GRUB parolası koyun ve disk şifrelemesi kullanın; root'u kilitlemek tek başına
fiziksel erişime karşı koruma sağlamaz.

---

## 7.8 Aşama 7 kontrol listesi

```bash
echo "--- servisler ---"
rc-update show default

echo "--- oturum ---"
ls /usr/share/xsessions/
update-alternatives --list x-session-manager

echo "--- ses ---"
dpkg -l pulseaudio rtkit | grep '^ii'
ls /etc/init.d/pulseaudio-enable-autospawn

echo "--- ag ---"
ls /etc/init.d/network-manager

echo "--- flatpak ---"
flatpak remotes -d 2>/dev/null || echo "ilk açılışta tekrar deneyin"

echo "--- kullanici ---"
id "$USERNAME"
passwd -S root
```

`rc-update show default` çıktısında en az şunlar olmalı:

```
dbus | default
elogind | default
cron | default
rsyslog | default
network-manager | default
lightdm | default
pulseaudio-enable-autospawn | default
snapper-boot | default
grub-btrfsd | default
```

Ve `boot` runlevel'ında:

```
zramswap | boot
```

---

➡️ Sonraki: [Aşama 8 — Temizlik ve Kurulumun Tamamlanması](08-temizlik-reboot.md)
