# Aşama 1 — Arch Live ISO Ortamının Hazırlanması

Amaç: Arch Live ISO içinde, Devuan Excalibur'u bootstrap edebilecek **doğru**
araçları hazırlamak ve Devuan deposunun imzasını uçtan uca doğrulamak.

---

## 1.1 Ön kontroller

```bash
# UEFI modunda mıyız? Bu dizin yoksa makine BIOS/Legacy modda açılmıştır.
ls /sys/firmware/efi/efivars >/dev/null && echo "UEFI: OK" || echo "UYARI: Legacy BIOS modu"

# Ağ bağlantısı
ping -c 3 pkgmaster.devuan.org

# Saat (GPG imza doğrulaması saat sapmasında başarısız olur)
timedatectl set-ntp true
timedatectl status | head -5
```

> Arch Live ISO'nun kendisi systemd kullanır; bu sorun değil. systemd'siz olan,
> kuracağımız **hedef sistem**dir.

---

## 1.2 Arch tarafındaki araçların kurulumu

```bash
pacman -Sy --needed --noconfirm \
    debootstrap arch-install-scripts \
    btrfs-progs dosfstools gptfdisk \
    wget curl gnupg binutils
```

| Paket | Ne için |
|---|---|
| `debootstrap` | Bağımlılıkları (perl, wget) için kurulur; **script'lerini kullanmayacağız** |
| `arch-install-scripts` | `arch-chroot` (bind-mount'ları otomatik yönetir) |
| `btrfs-progs` | `mkfs.btrfs`, `btrfs subvolume` |
| `dosfstools` | `mkfs.vfat` (EFI bölümü) |
| `gptfdisk` | `sgdisk` (GPT bölümleme) |
| `binutils` | `ar` — `.deb` arşivlerini açmak için (Arch'ta `dpkg-deb` yok) |
| `gnupg` | `gpg` / `gpgv` |

---

## 1.3 ⚠️ Neden Arch'ın `debootstrap` script'ini kullanmıyoruz?

Debian'ın `debootstrap` paketi Devuan süitlerini **içermez**. İnternette çok
yaygın olan şu öneri **hatalıdır**:

```bash
# YAPMAYIN — bu kurulumu bozar:
# ln -s /usr/share/debootstrap/scripts/sid /usr/share/debootstrap/scripts/excalibur
```

Devuan'ın `excalibur` script'i (aslında `ceres`'e symlink) Debian'ın `sid`
script'inden şu kritik noktalarda ayrılır:

- Anahtarlık olarak `devuan-archive-keyring.gpg` kullanır (Debian'ınkini değil).
- Taban pakete `devuan-keyring` **ve `sysvinit-core`**'u zorla ekler.
- `excalibur`/`freia`/`ceres` için merged-`/usr`'ı zorunlu kılar, `usr-is-merged`
  paketini ekler ve `usrmerge`'i dışlar.

`sid` script'iyle bootstrap edilen sistem `sysvinit-core` olmadan gelir — yani
init sistemi olmayan, açılmayan bir taban. Bu yüzden Devuan'ın kendi
`debootstrap` paketini indirip kullanacağız.

---

## 1.4 Devuan anahtarlığı ve debootstrap paketinin indirilmesi

Sürüm numaraları zamanla değişir; bu yüzden dosya adlarını paket indeksinden
otomatik çözüyoruz.

```bash
export MIRROR="http://pkgmaster.devuan.org/merged"
export SUITE="excalibur"
export WORK="/root/devuan-bootstrap"

mkdir -p "$WORK" && cd "$WORK"

# Paket indeksi
curl -fsSL "$MIRROR/dists/$SUITE/main/binary-amd64/Packages.gz" -o Packages.gz
gzip -dkf Packages.gz          # -> ./Packages

# Havuz yolu ve SHA256'yı indeksten okuyan yardımcılar
deb_path() { awk -v p="Package: $1" '$0==p{f=1} f&&/^Filename:/{print $2; exit}' Packages; }
deb_hash() { awk -v p="Package: $1" '$0==p{f=1} f&&/^SHA256:/{print $2; exit}' Packages; }

curl -fsSL -o devuan-keyring.deb "$MIRROR/$(deb_path devuan-keyring)"
curl -fsSL -o debootstrap.deb    "$MIRROR/$(deb_path debootstrap)"

ls -la *.deb
```

2026-08-02 itibarıyla çözülen yollar (sizde farklı olabilir, normaldir):

```
pool/DEVUAN/main/d/devuan-keyring/devuan-keyring_2025.08.09_all.deb
pool/DEVUAN/main/d/debootstrap/debootstrap_1.0.141devuan1_all.deb
```

---

## 1.5 GPG güven zincirinin kurulması

Zincir şöyle: **parmak izi (elle doğrulanır)** → **InRelease imzası** →
**Packages.gz özeti** → **`.deb` özetleri**.

### Adım 1 — Anahtarlığı aç

```bash
cd "$WORK"
mkdir -p keyring && (cd keyring && ar x ../devuan-keyring.deb && tar -xf data.tar.*)

export DEVUAN_KEYRING="$WORK/keyring/usr/share/keyrings/devuan-archive-keyring.pgp"
ls -la "$DEVUAN_KEYRING"
```

### Adım 2 — Excalibur imza anahtarının parmak izini doğrula

```bash
gpg --show-keys --with-fingerprint --with-colons "$DEVUAN_KEYRING" 2>/dev/null \
  | awk -F: '/^fpr/{print $10}' | grep -x 9F8D6C74DE661075FD171BE3B3982868D104092C \
  && echo "Parmak izi DOĞRU" || echo "!!! PARMAK İZİ EŞLEŞMEDİ — DURUN !!!"
```

Beklenen anahtar:

```
pub   rsa4096 2022-09-22 [SC]
      9F8D 6C74 DE66 1075 FD17  1BE3 B398 2868 D104 092C
uid   Devuan Release Signing (Excalibur) <repository@devuan.org>
```

> Bu parmak izini bağımsız bir kanaldan (devuan.org, başka bir makine) da
> teyit etmeniz önerilir — güven zincirinin tek elle doğrulanan halkası budur.

### Adım 3 — Depo imzasını doğrula

```bash
cd "$WORK"
curl -fsSL "$MIRROR/dists/$SUITE/InRelease" -o InRelease
gpgv --keyring "$DEVUAN_KEYRING" InRelease
```

Beklenen çıktı:

```
gpgv: Signature made ...
gpgv:                using RSA key 9F8D6C74DE661075FD171BE3B3982868D104092C
gpgv: Good signature from "Devuan Release Signing (Excalibur) <repository@devuan.org>"
```

`InRelease` başlığı ayrıca süiti teyit eder:

```bash
head -12 InRelease
# Origin: Devuan / Suite: stable / Version: 6.0 / Codename: excalibur
```

### Adım 4 — Paket indeksinin özetini doğrula

```bash
want=$(awk '/^SHA256:/{f=1;next} f&&$3=="main/binary-amd64/Packages.gz"{print $1; exit}' InRelease)
have=$(sha256sum Packages.gz | cut -d' ' -f1)
[ "$want" = "$have" ] && echo "Packages.gz: OK" || echo "!!! Packages.gz UYUŞMUYOR !!!"
```

### Adım 5 — İndirilen `.deb`'lerin özetlerini doğrula

```bash
check_deb() {   # $1=paket adı  $2=yerel dosya
  local want have
  want=$(deb_hash "$1"); have=$(sha256sum "$2" | cut -d' ' -f1)
  if [ "$want" = "$have" ]; then echo "$1: OK"; else echo "!!! $1: SHA256 UYUŞMUYOR !!!"; return 1; fi
}

check_deb devuan-keyring devuan-keyring.deb
check_deb debootstrap    debootstrap.deb
```

Beş adım da OK verirse depo ve indirilenler doğrulanmıştır.

---

## 1.6 Devuan debootstrap'ının hazırlanması

Arch'ın kendi kurulumunu **bozmadan**, Devuan'ın script'lerini ayrı bir dizinde
kullanıyoruz.

```bash
cd "$WORK"
mkdir -p dbs && (cd dbs && ar x ../debootstrap.deb && tar -xf data.tar.*)

# Devuan'ın excalibur script'i gerçekten var mı?
ls -l dbs/usr/share/debootstrap/scripts/excalibur
# -> excalibur -> ceres  (symlink)

# Bundan sonra kullanacağımız değişkenler
export DEBOOTSTRAP_DIR="$WORK/dbs/usr/share/debootstrap"
export DEBOOTSTRAP_BIN="$WORK/dbs/usr/sbin/debootstrap"
chmod +x "$DEBOOTSTRAP_BIN"
```

`ceres` script'i anahtarlığı sabit `/usr/share/keyrings/devuan-archive-keyring.gpg`
yolundan arar. Hem oraya kopyalıyor hem de `--keyring` ile açıkça geçiyoruz:

```bash
install -Dm644 "$WORK/keyring/usr/share/keyrings/devuan-archive-keyring.pgp" \
               /usr/share/keyrings/devuan-archive-keyring.pgp
ln -sf devuan-archive-keyring.pgp /usr/share/keyrings/devuan-archive-keyring.gpg
```

Doğrulama:

```bash
DEBOOTSTRAP_DIR="$DEBOOTSTRAP_DIR" "$DEBOOTSTRAP_BIN" --version
# -> debootstrap 1.0.141devuan1
```

---

## 1.7 Ortam değişkenlerini kalıcılaştırma

Live ISO'da yeni bir kabuk açarsanız bunlar kaybolur. Tek dosyada toplayın:

```bash
cat > /root/devuan-env.sh <<EOF
export MIRROR="$MIRROR"
export SUITE="$SUITE"
export WORK="$WORK"
export DEVUAN_KEYRING="$DEVUAN_KEYRING"
export DEBOOTSTRAP_DIR="$DEBOOTSTRAP_DIR"
export DEBOOTSTRAP_BIN="$DEBOOTSTRAP_BIN"
EOF

# Yeni kabukta:  source /root/devuan-env.sh
```

---

➡️ Sonraki: [Aşama 2 — Disk Bölümleme ve BTRFS Subvolume Mimarisi](02-disk-btrfs.md)
