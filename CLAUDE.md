# PROMPT: Arch Live ISO Üzerinden Debootstrap ile Devuan Excalibur (OpenRC) Kurulum Rehberi

Sen kıdemli bir Linux Sistem Yöneticisi ve dağıtım mimarısın. 
Görevin, **Arch Linux Live ISO** ortamındayken `debootstrap` kullanarak hedef diske sıfırdan **Devuan Excalibur** (systemd-free, OpenRC) işletim sistemini kurmam için bana adım adım, hatasız ve detaylı bir rehber oluşturmaktır.

Aşağıdaki teknik gereksinimleri ve mimari mimari kriterleri **eksiksiz** karşılayan, komut bloğu odaklı ve açıklamalı bir rehber sunmanı istiyorum.

---

## 🛠 Mimari ve Paket Gereksinimleri

1. **Init Sistemi:** 
   - Tamamen `systemd` barındırmayan saf **OpenRC** tabanlı yapı.
   - Oturum ve yetkiler için `elogind` ve `eudev` entegrasyonu.

2. **Çekirdek (Kernel):**
   - Sisteme **2 adet yedekli LTS Çekirdek** kurulmalıdır (Örn: Ana `linux-image-amd64` ve ek yedek LTS çekirdeği).

3. **Dosya Sistemi & Snapshot Yönetimi (BTRFS Stack):**
   - Kök dizinde **BTRFS** kullanımı.
   - Doğru Subvolume düzeni (`@`, `@home`, `@snapshots`, `@var_log`).
   - `apt` işlemleri öncesi/sonrası otomatik snapshot alan APT hook entegrasyonu (`snapper-apt-plugin` / `80snapper`).
   - Bakım ve snapshot araçları: `snapper`, `snapper-gui`, `btrfs-assistant`, `btrfsmaintenance`.
   - Boot menüsünden snapshot'lara dönebilmek için **GRUB** ve **`grub-btrfs`** yapılandırması (OpenRC uyumlu daemon/update mekanizmasıyla).

4. **Bellek & Sistem Optimizasyonu:**
   - **zram** yapılandırması (`zram-tools` veya OpenRC uyumlu zram servisi).

5. **Masaüstü ve Ses:**
   - **XFCE** masaüstü ortamı ve **LightDM** display manager.
   - Ses mimarisi olarak doğrudan **PulseAudio** kullanımı.
   - **Wayland** istemiyorum.

6. **Paket Yönetimi:**
   - **Flatpak** ve Flathub deposu entegrasyonu.

---

## 📋 Adım Adım Kurulum Akışı (Rehber Formatı)

Lütfen rehberi aşağıdaki ana aşamalara bölerek, her komutun ne işe yaradığını kısaca açıklayacak şekilde hazırla:

### Aşama 1: Arch Live ISO Ortamının Hazırlanması
- Arch ISO üzerinde `debootstrap` ve gerekli disk araçlarının kurulması (`pacman -Sy debootstrap arch-install-scripts btrfs-progs`).
- Devuan depolarına erişim ve GPG imza doğrulama adımları.

### Aşama 2: Disk Bölümleme ve BTRFS Subvolume Mimarisi
- Disklerin (EFI: FAT32, ROOT: BTRFS) biçimlendirilmesi.
- / = nvme1n1 /home = nvme0n1 gibi olacak. HOME dizini AYRI disk.
- BTRFS Subvolume'larının oluşturulması:
  - `@` -> `/`
  - `@home` -> `/home`
  - `@snapshots` -> `/.snapshots`
  - `@var_log` -> `/var/log`
- Subvolume'ların uygun mount seçenekleri (`noatime,compress=zstd:1,space_cache=v2,subvol=...`) ile `/mnt` altına bağlanması.
- fstrim.timer kullanamayacagimiz icin btrfs default strim olsun.

### Aşama 3: Debootstrap ile Devuan Excalibur İndirme
- `debootstrap` komutu ile `excalibur` (Debian 13 Trixie karşılığı) tabanının `/mnt` dizinine çekilmesi:
  `http://pkgmaster.devuan.org/merged excalibur`

### Aşama 4: Chroot Ortamına Geçiş ve Temel OpenRC Yapılandırması
- Mount işlemlerinin (`/dev`, `/proc`, `/sys`, `/run`) tamamlanıp chroot'a girilmesi.
- Devuan `sources.list` dosyasının yapılandırılması.
- OpenRC, `sysvinit-core`, `elogind`, `eudev` paketlerinin ana init sistemi olarak kurulması ve systemd paketlerinin kalıcı olarak yasaklanması (`apt-mark` veya `apt/preferences.d`).

### Aşama 5: Çekirdek, Bootloader ve BTRFS Yapılandırması
- 2 adet LTS Kernel paketinin kurulması ve yedekli boot yapısının oluşturulması.
- GRUB (UEFI) kurulumu ve `grub-btrfs` entegrasyonu.
- `/etc/fstab` dosyasının UUID'ler ile oluşturulması.

### Aşama 6: Snapper, APT Hook ve Bakım Araçları
- `snapper` yapılandırmasının oluşturulması.
- APT işlemleri öncesinde otomatik snapshot alma mekanizmasının kurulması.
- `snapper-gui`, `btrfs-assistant` ve `btrfsmaintenance` paketlerinin kurulumu ve OpenRC altında çalışacak şekilde ayarlanması.
- OpenRC için `zram` yapılandırmasının yapılması.

### Aşama 7: XFCE, PulseAudio, Flatpak ve Kullanıcı Ayarları
- XFCE4, LightDM, NetworkManager kurulumu ve OpenRC servislerine eklenmesi (`rc-update add ...`).
- PulseAudio kurulumu ve yetkilendirmesi.
- Flatpak ve Flathub reposunun eklenmesi.
- Sudo yetkili kullanıcı ve root parolası tanımları.
- Root hesabi kilitlensin, kullanmayacagiz.

### Aşama 8: Temizlik ve Kurulumun Tamamlanması
- Chroot ortamından çıkış, unmount adımları ve sistemi güvenle yeniden başlatma talimatları.

---

**Not:** Komut bloklarında belirsiz parametreler (örneğin `/dev/sdX` veya `/dev/nvme0n1pX`) kullanırken beni uyarmak için yorum satırları ekle. Tüm servis komutlarını `systemctl` yerine OpenRC (`rc-update`, `rc-service`) yapısına uygun yaz.
