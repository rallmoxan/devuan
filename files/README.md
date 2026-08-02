# `files/` — Hazır Yapılandırma Dosyaları

Rehberde heredoc olarak da verilen, elle yazılmış dosyalar. Devuan'ın
paketlerinde bulunmadıkları için buradalar.

| Dosya | Hedef yol | Neden gerekli |
|---|---|---|
| `zramswap.initd` | `/etc/init.d/zramswap` | `zram-tools` paketi yalnızca bir systemd unit'i getirir, init script'i yoktur |
| `snapper-boot.initd` | `/etc/init.d/snapper-boot` | `snapper` paketi yalnızca systemd unit/timer getirir |
| `snapper-timeline.cron` | `/etc/cron.hourly/snapper-timeline` | `snapper-timeline.timer` (OnCalendar=hourly) karşılığı |
| `snapper-cleanup.cron` | `/etc/cron.daily/snapper-cleanup` | `snapper-cleanup.timer` karşılığı |
| `no-systemd.pref` | `/etc/apt/preferences.d/no-systemd.pref` | systemd'nin geri sızmasına karşı APT pin'i |
| `openrc-sync-services` | `/usr/local/sbin/openrc-sync-services` | `update-rc.d` OpenRC runlevel'larını güncellemez |

## Toplu kurulum

Chroot içinde, depo `/mnt/root/devuan-guide` altına kopyalanmışsa:

```bash
SRC=/root/devuan-guide/files

install -m 755 "$SRC/zramswap.initd"          /etc/init.d/zramswap
install -m 755 "$SRC/snapper-boot.initd"      /etc/init.d/snapper-boot
install -m 755 "$SRC/snapper-timeline.cron"   /etc/cron.hourly/snapper-timeline
install -m 755 "$SRC/snapper-cleanup.cron"    /etc/cron.daily/snapper-cleanup
install -m 644 "$SRC/no-systemd.pref"         /etc/apt/preferences.d/no-systemd.pref
install -m 755 "$SRC/openrc-sync-services"    /usr/local/sbin/openrc-sync-services

rc-update add zramswap    boot
rc-update add snapper-boot default

rc-update show
```

> `.initd` / `.cron` uzantıları yalnızca bu depodaki dosyaları ayırt etmek
> içindir. **Hedef yola kopyalarken uzantıyı kaldırın** — `run-parts`, nokta
> içeren dosya adlarını `/etc/cron.*` altında çalıştırmaz.

## Kaynaklar

Bu dosyalardaki komutlar tahmin değildir; ilgili paketlerin içindeki systemd
unit'lerinden birebir alınmıştır:

```
snapper-timeline.service : ExecStart=/usr/lib/snapper/systemd-helper --timeline
snapper-cleanup.service  : ExecStart=/usr/lib/snapper/systemd-helper --cleanup
snapper-boot.service     : ExecStart=/usr/bin/snapper --config root create \
                                     --cleanup-algorithm number --description "boot"
zramswap.service         : /usr/sbin/zramswap start|stop  (paketin kendi script'i)
```
