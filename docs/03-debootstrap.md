# Aşama 3 — Debootstrap ile Devuan Excalibur İndirme

Bu aşamada `/mnt` altına minimum Devuan Excalibur taban sistemi kurulur.

---

## 3.1 Ön kontrol

Yeni bir kabuk açtıysanız değişkenleri geri yükleyin ve mount düzenini
doğrulayın:

```bash
source /root/devuan-env.sh
findmnt -R /mnt

# Devuan debootstrap'ı hazır mı?
DEBOOTSTRAP_DIR="$DEBOOTSTRAP_DIR" "$DEBOOTSTRAP_BIN" --version
ls -l "$DEBOOTSTRAP_DIR/scripts/excalibur"     # -> ceres symlink'i
```

---

## 3.2 Bootstrap

```bash
DEBOOTSTRAP_DIR="$DEBOOTSTRAP_DIR" "$DEBOOTSTRAP_BIN" \
    --arch=amd64 \
    --merged-usr \
    --components=main,contrib,non-free,non-free-firmware \
    --keyring="$DEVUAN_KEYRING" \
    --include=ca-certificates,gnupg,apt-utils,dialog,locales \
    excalibur \
    /mnt \
    "$MIRROR"
```

| Parametre | Açıklama |
|---|---|
| `--arch=amd64` | Hedef mimari |
| `--merged-usr` | `/bin`, `/sbin`, `/lib` → `/usr/...` symlink'leri. Excalibur için **zorunlu**; script `MERGED_USR=no` görürse hata verir |
| `--components=...` | `non-free-firmware` olmadan Wi-Fi/GPU firmware'ı çekilemez |
| `--keyring=` | Aşama 1'de parmak izi doğrulanan anahtarlık |
| `--include=` | Chroot'a girer girmez gereken minimum ek; gerisi Aşama 4+'ta |

Süre bağlantıya göre 3–15 dakika, indirilen boyut ~250 MB, disk üzerinde ~600 MB.

### `--include` listesi neden bu kadar kısa?

`debootstrap` bağımlılık çözümlemesi `apt` kadar akıllı değildir (alternatif
bağımlılıkları çözemez). Ne kadar çok paket eklerseniz taban sistemin yarıda
kırılma ihtimali o kadar artar. Gerçek paket kurulumunu, tam işlevli `apt`'ın
bulunduğu chroot içinde yapacağız.

### Devuan script'inin sessizce eklediği paketler

`ceres` script'i taban sete şunları **kendiliğinden** ekler — sizin
belirtmenize gerek yok:

```
devuan-keyring      # depo anahtarları hedef sisteme kurulur
sysvinit-core       # PID 1 (Debian'ın sid script'i bunu YAPMAZ)
usr-is-merged       # merged-/usr geçişinin tamamlandığını işaretler
```

---

## 3.3 Doğrulama

```bash
# Sürüm işaretleri
cat /mnt/etc/devuan_version        # -> excalibur
cat /mnt/etc/debian_version        # -> 13.x
grep PRETTY /mnt/usr/lib/os-release
# -> PRETTY_NAME="Devuan GNU/Linux 6 (excalibur)"

# PID 1 sağlayıcısı kuruldu mu?
chroot /mnt dpkg -l sysvinit-core | tail -1

# systemd bulaşmış mı? (hiçbir satır dönmemeli)
chroot /mnt dpkg -l | grep -E '^ii\s+(systemd|systemd-sysv|libpam-systemd)\s' || echo "systemd YOK — beklenen sonuç"

# merged-/usr doğru mu?
ls -ld /mnt/bin /mnt/sbin /mnt/lib      # üçü de -> usr/... symlink'i olmalı
```

> `libsystemd0` paketinin kurulu **olması normaldir**. Bu yalnızca bir kütüphane
> shim'idir; init sistemi değildir ve PulseAudio dahil pek çok paket ona bağlıdır.
> Kaldırmaya çalışmayın.

---

## 3.4 Hata durumunda

**`E: No such script: .../excalibur`**
→ `DEBOOTSTRAP_DIR` Devuan'ın dizinini göstermiyor. Aşama 1.6'yı tekrarlayın.

**`E: Release signed by unknown key`**
→ `--keyring` yolu yanlış ya da anahtarlık açılmamış. Aşama 1.5'i tekrarlayın.

**`E: Couldn't find these debs: ...`**
→ Ayna geçici olarak tutarsız. Başka bir aynayı deneyin:

```bash
export MIRROR="http://deb.devuan.org/merged"     # round-robin ayna havuzu
```

**Baştan başlamak gerekirse:**

```bash
# ⚠️ /mnt altındaki her şey silinir; bağlı disklerin doğru olduğunu teyit edin
findmnt -R /mnt
umount -R /mnt
mount -o "${BTRFS_OPTS},subvol=@" "$ROOT_PART" /mnt
find /mnt -mindepth 1 -delete
# ardından 2.6'daki mount adımlarını ve bu aşamayı tekrarlayın
```

---

➡️ Sonraki: [Aşama 4 — Chroot ve Temel OpenRC Yapılandırması](04-chroot-openrc.md)
