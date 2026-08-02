# Arch Live ISO → Devuan Excalibur (OpenRC) — Debootstrap Kurulum Rehberi

Bu depo, **Arch Linux Live ISO** ortamından `debootstrap` ile hedef diske sıfırdan
**Devuan 6.0 "Excalibur"** (systemd'siz, OpenRC tabanlı) kurmak için adım adım bir
rehber içerir.

Rehberdeki paket adları, sürümler, init script'lerin varlığı ve dosya yolları
**tahmin edilmemiş**, `pkgmaster.devuan.org` üzerindeki gerçek Excalibur paket
indeksinden ve paketlerin içinden doğrulanmıştır (doğrulama tarihi: **2026-08-02**).
Doğrulanan ayrıntılar için [Doğrulanmış Gerçekler](#-doğrulanmış-gerçekler)
bölümüne bakın.

---

## 🎯 Hedef Mimari

| Katman | Seçim |
|---|---|
| Dağıtım | Devuan 6.0 Excalibur (Debian 13 Trixie tabanı) |
| PID 1 | `sysvinit-core` |
| Servis yöneticisi | `openrc` (`sysv-rc` yerine geçer) |
| Oturum / yetki | `elogind` + `libpam-elogind` + `polkitd` |
| Aygıt yöneticisi | `eudev` |
| Kök dosya sistemi | BTRFS (`@`, `@snapshots`, `@var_log`) — `/dev/nvme1n1` |
| `/home` | BTRFS (`@home`) — ayrı disk, `/dev/nvme0n1` |
| Çekirdek | 2 adet yedekli LTS 6.12.x çekirdeği |
| Önyükleyici | GRUB (UEFI) + kaynaktan derlenmiş `grub-btrfs` (OpenRC modu) |
| Snapshot | `snapper` + paketle gelen `80snapper` APT hook'u |
| Bakım | `btrfsmaintenance` (cron modu), `btrfs-assistant`, `snapper-gui` |
| Bellek | `zram-tools` + elle yazılmış OpenRC init script'i |
| TRIM | BTRFS `discard=async` (fstrim.timer yok) |
| Masaüstü | XFCE 4.20 + LightDM + Xorg (**Wayland yok**) |
| Ses | PulseAudio 17 (PipeWire yok) |
| Ek paket yönetimi | Flatpak + Flathub |

---

## 📋 Kurulum Aşamaları

| # | Aşama | Dosya |
|---|---|---|
| 1 | Arch Live ISO ortamının hazırlanması, Devuan anahtarlığı ve GPG doğrulama | [docs/01-arch-live-hazirlik.md](docs/01-arch-live-hazirlik.md) |
| 2 | Disk bölümleme ve BTRFS subvolume mimarisi | [docs/02-disk-btrfs.md](docs/02-disk-btrfs.md) |
| 3 | Debootstrap ile Excalibur tabanının indirilmesi | [docs/03-debootstrap.md](docs/03-debootstrap.md) |
| 4 | Chroot ve temel OpenRC yapılandırması | [docs/04-chroot-openrc.md](docs/04-chroot-openrc.md) |
| 5 | Çekirdek, GRUB, fstab | [docs/05-kernel-grub-fstab.md](docs/05-kernel-grub-fstab.md) |
| 6 | Snapper, APT hook, grub-btrfs, btrfsmaintenance, zram | [docs/06-snapper-bakim-zram.md](docs/06-snapper-bakim-zram.md) |
| 7 | XFCE, LightDM, PulseAudio, Flatpak, kullanıcılar | [docs/07-xfce-ses-flatpak.md](docs/07-xfce-ses-flatpak.md) |
| 8 | Temizlik, unmount, yeniden başlatma | [docs/08-temizlik-reboot.md](docs/08-temizlik-reboot.md) |
| + | Ek: rollback, sorun giderme, saf `openrc-init` | [docs/09-ek-rollback-sorun-giderme.md](docs/09-ek-rollback-sorun-giderme.md) |

Hazır dosyalar (rehber içinde heredoc olarak da veriliyor): [`files/`](files/)

---

## ⚠️ Başlamadan Önce

> **VERİ KAYBI UYARISI.** Rehberdeki disk aygıtı adları (`/dev/nvme0n1`,
> `/dev/nvme1n1`) **örnektir**. `mkfs`, `sgdisk` ve `wipefs` komutları hedefteki
> her şeyi geri dönüşsüz siler. Her komut bloğunda aygıt adı geçen satırların
> başına uyarı yorumu konmuştur; komutu çalıştırmadan önce `lsblk -o
> NAME,SIZE,MODEL,SERIAL` çıktısıyla doğrulayın.

Gereksinimler:

- UEFI modunda başlatılmış Arch Linux Live ISO (`ls /sys/firmware/efi` çıktı vermeli)
- Çalışan internet bağlantısı
- İki NVMe disk (biri kök, biri `/home`) — tek diskle de yapılabilir, bkz. Aşama 2 notu
- x86_64 (amd64) donanım

---

## ✅ Doğrulanmış Gerçekler

Bu rehberin dayandığı, Excalibur deposundan **doğrudan doğrulanmış** bulgular.
Çoğu rehberin yanlış yaptığı noktalar burada:

### 1. Arch'ın `debootstrap`'ı Excalibur'u tanımaz — `sid`'e symlink atmak YANLIŞTIR

Debian'ın `debootstrap` paketi Devuan süitlerini içermez. Yaygın "çözüm" olan
`ln -s /usr/share/debootstrap/scripts/sid .../excalibur` **hatalıdır**, çünkü
Devuan'ın `ceres` script'i Debian'ın `sid` script'inden esaslı biçimde farklıdır:

```
# Devuan ceres script'inden (excalibur -> ceres symlink'i):
keyring /usr/share/keyrings/devuan-archive-keyring.gpg
devuan_required="devuan-keyring sysvinit-core"      # tabana zorla eklenir
case "$CODENAME" in excalibur|freia|ceres)          # merged-/usr zorunlu
    if [ "$MERGED_USR" = "no" ]; then error 1 ...
```

`sid` script'i kullanılırsa taban sistem **`sysvinit-core` olmadan** kurulur ve
merged-`/usr` denetimi yapılmaz. Bu yüzden Aşama 1'de Devuan'ın kendi
`debootstrap` paketi indirilip `DEBOOTSTRAP_DIR` ile kullanılır.

### 2. Excalibur deposunda `systemd` paketi **hiç yok**

`systemd`, `systemd-sysv`, `libpam-systemd` — üçü de Devuan merged deposunda
mevcut değil (amprolla bunları ayıklıyor). `libpam-elogind` zaten
`Provides: libpam-systemd, logind, default-logind` sağlıyor. Yine de Aşama 4'te
bir APT pin'i konur; bu ileride bir Debian deposu eklenirse devreye girecek
emniyet kemeridir. `libsystemd0` bir istisnadır: PulseAudio dahil pek çok paket
ona bağlıdır, yasaklanmamalıdır.

### 3. `snapper` paketi APT hook'unu **zaten getiriyor**

Ayrıca bir `snapper-apt-plugin` kurmaya gerek yok. `snapper 0.10.6-1.2` paketi
`/etc/apt/apt.conf.d/80snapper` dosyasını içeriyor ve şu koşullarla çalışıyor:

- `/usr/bin/snapper` var
- `/etc/default/snapper` içinde `DISABLE_APT_SNAPSHOT` ≠ `yes`
- **`/etc/snapper/configs/root` mevcut**

Yani hook, `snapper -c root create-config /` çalıştırıldığı anda kendiliğinden
aktifleşir (Aşama 6).

### 4. `grub-btrfs` Debian/Devuan'da paketlenmemiştir

`excalibur`, `excalibur-backports`, `excalibur-proposed-updates` ve `ceres`
süitlerinin hiçbirinde `grub-btrfs` yok. Kaynaktan derlenmelidir — iyi haber:
üstkaynak Makefile **yerleşik OpenRC desteği** sunuyor:

```
make OPENRC=true SYSTEMD=false install
```

Bu, `/etc/init.d/grub-btrfsd` ve `/etc/conf.d/grub-btrfsd` kurar. `grub-btrfsd`
çalışmak için `inotifywait` ister → `inotify-tools` paketi gerekir.

### 5. `snapper` ve `zram-tools` OpenRC init script'i getirmez

| Paket | `/etc/init.d/` script'i | Çözüm |
|---|---|---|
| `elogind` | ✅ `elogind` | — |
| `eudev` | ✅ `eudev` | — |
| `dbus` | ✅ `dbus` | — |
| `cron` | ✅ `cron` | — |
| `lightdm` | ✅ `lightdm` | — |
| `network-manager` | ✅ `network-manager` | — |
| `snapper` | ❌ (yalnız systemd unit/timer) | cron + elle init script (Aşama 6) |
| `zram-tools` | ❌ (yalnız systemd unit) | elle init script (Aşama 6) |
| `btrfsmaintenance` | ❌ (systemd unit) | paketin `...-refresh-cron.sh cron` modu |

`snapperd` D-Bus ile aktive olur (`/usr/share/dbus-1/system-services/org.opensuse.Snapper.service`),
bu yüzden systemd olmadan da sorunsuz çalışır.

### 6. BTRFS `discard=async` zaten çekirdek varsayılanıdır

btrfs-progs belgelerinden: *"(default: async when devices support it since 6.2)"*.
Kurulacak çekirdek 6.12 LTS olduğu için `fstrim.timer` olmaması bir sorun değil.
Yine de fstab'a açıkça yazılır ve `btrfsmaintenance` içinde
`BTRFS_TRIM_PERIOD="none"` bırakılır ki iki mekanizma çakışmasın.

### 7. XFCE, PipeWire'ı sürüklemez

`xfce4` metapaketi PipeWire'a bağlı değil. `xfce4-pulseaudio-plugin` yalnızca
`Recommends: pulseaudio | pipewire-pulse` diyor ve ilk alternatif PulseAudio.
`pulseaudio` paketini açıkça kurmak yeterli.

### 8. Doğrulanmış paket sürümleri (2026-08-02, excalibur/main amd64)

| Paket | Sürüm | | Paket | Sürüm |
|---|---|---|---|---|
| `openrc` | 0.56-1 | | `snapper` | 0.10.6-1.2 |
| `sysvinit-core` | 3.14-4devuan1 | | `snapper-gui` | 0git.960a94834f-6.1 |
| `elogind` | 255.17-2 | | `btrfs-assistant` | 2.1.1-1+b1 |
| `eudev` | 3.2.14-4 | | `btrfsmaintenance` | 0.5.2-1 |
| `linux-image-amd64` | 6.12.94-1 | | `zram-tools` | 0.3.7-1 |
| `xfce4` | 4.20.1 | | `flatpak` | 1.16.6-1~deb13u1 |
| `lightdm` | 1.32.0-6devuan1+excalibur2 | | `pulseaudio` | 17.0+dfsg1-2+b1 |

### 9. Excalibur imza anahtarı

```
pub   rsa4096 2022-09-22 [SC]
      9F8D 6C74 DE66 1075 FD17  1BE3 B398 2868 D104 092C
uid   Devuan Release Signing (Excalibur) <repository@devuan.org>
```

`dists/excalibur/InRelease` bu anahtarla imzalıdır (`Suite: stable`,
`Version: 6.0`, `Codename: excalibur`). Aşama 1 bu zinciri uçtan uca doğrular.

---

## 🔁 Kurulumdan Sonra

- Snapshot'a geri dönüş: [docs/09-ek-rollback-sorun-giderme.md](docs/09-ek-rollback-sorun-giderme.md)
- GRUB menüsünde **"Devuan snapshots"** alt menüsü `grub-btrfsd` tarafından
  otomatik güncellenir; elle tetiklemek için `rc-service grub-btrfsd restart`.
