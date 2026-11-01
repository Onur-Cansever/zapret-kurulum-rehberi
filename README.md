# zapret Kurulum Rehberi

SNI-tabanlı DPI engellerini aşmak için **zapret**'in agresif ClientHello desync (senkronizasyon kaybı) özelliğini kullanan, yerinde derlenip kendi makinenizde çalışan bir SOCKS5 proxy kurulumu.

> **Neden zapret?** Sıradan bir SOCKS proxy (sing-box vb.) DPI'ın SNI'yi okumasını engellemez; zapret, TLS ClientHello'yu bölerek ve karıştırarak (desync) DPI'nin SNI'yi tanımasını imkansız kılar.

## Gereksinimler

- **C derleyici:** `gcc` ve `make`
- **Kaynak yönetim aracı:** `git`
- **OS:** Linux (test edildi: Gentoo, Debian tabanlı dağıtımlar)
- **Hedef:** `libnetfilter_queue` gerektirmez (SOCKS modeli kullanıcı uzayında çalışır)

## 1. Kaynak İndirme ve Derleme

```bash
git clone https://github.com/bol-van/zapret.git
cd zapret
make
```

Derleme sonrası `tpws/tpws`, `ip2net/ip2net`, `mdig/mdig` binaryleri oluşur.
`nfqws` (NFQUEUE modeli) `libnetfilter_queue` dev header'ları ister; SOCKS modeli için **gerekli değildir**.

## 2. tpws'i (SOCKS5 Proxy) Çalıştırma

Aşağıdaki komut agresif desync parametreleriyle tpws'i başlatır:

```bash
./tpws/tpws \
  --socks \
  --port=20090 \
  --bind-addr=0.0.0.0 \
  --filter-tcp=443 \
  --split-pos=1,midsld \
  --disorder \
  --oab=tls \
  --tlsrec=1 \
  --hostlist-auto=/home/KULLANICI/.../zapret/auto-hostlist.txt \
  --hostlist-auto-fail-threshold=1 \
  --debug=@$(pwd)/tpws-debug.log
```

### Parametreler

| Parametre | Açıklama |
|-----------|----------|
| `--socks` | SOCKS5 proxy modu |
| `--port=20090` | Dinlenecek port (>1024, sıradan kullanıcı için) |
| `--bind-addr=0.0.0.0` | LAN'a da açmak için; sadece lokal için `127.0.0.1` |
| `--filter-tcp=443` | Sadece TCP/443 (TLS) trafiği desync'lenir |
| `--split-pos=1,midsld` | ClientHello'nun ilk baytından sonra ve ortada bölünür |
| `--disorder` | ClientHello parçaları karıştırılır |
| `--oab=tls` | Out-of-band desync, TLS protokole uygulanır |
| `--tlsrec=1` | TLS replay koruması (zincirleme DPI'lara karşı) |
| `--hostlist-auto=...` | Otomatik hostlist dosyası — engellenmiş domainler otomatik eklenir |
| `--hostlist-auto-fail-threshold=1` | 1 başarısız bağlantıda domain hostlist'a eklenir |
| `--debug=@...` | Hata ayıklama logu (mutlak yol) |

> **Önemli:** Hostlist'teki domain'lerin trafiği desync'lenir; listede olmayan domain'ler normal (transparent) geçer. Bu yüzden hostlist'ı dar tutmanız, diğer sitelerinizi etkilemez.

### Hostlist Otomatik Algılama

`--hostlist-auto` ile zapret, proxy üzerinden gelen ve başarısız olan (RST, timeout) bağlantıları otomatik olarak hostlist dosyasına ekler. Yani yeni bir domain engellendiğinde elle ekleme gerekmez; zapret bir sonraki bağlantıda otomatik algılar.

## 3. OpenRC Servisi (Gentoo)

Gentoo ve OpenRC kullananlar için `/etc/init.d/zapret` servis dosyası:

```sh
#!/sbin/openrc-run

command="$(dirname $0)/../usr/local/bin/tpws"
command_args="--socks --port=20090 --bind-addr=127.0.0.1 --filter-tcp=443 \
  --split-pos=1,midsld --disorder --oab=tls --tlsrec=1 \
  --hostlist-auto=/etc/zapret/auto-hostlist.txt --hostlist-auto-fail-threshold=1"
command_background="true"
pidfile="/run/zapret.pid"
command_user="KULLANICI"
command_group="KULLANICI"
output_log="/var/log/zapret.log"
error_log="/var/log/zapret.log"

depend() {
  need net
}

start_pre() {
  checkpath -d -m 0644 -o KULLANICI:KULLANICI /var/log/zapret.log
  checkpath -a -m 0644 -o KULLANICI:KULLANICI /etc/zapret/auto-hostlist.txt
  [ -x "$command" ] || eerror "tpws bulunamadı: $command" && return 1
}
```

> `command_user`'i kendi kullanıcınızla değiştirin. `command`'ın yolu `tpws`'in konumuna göre ayarlanmalıdır.

Kurulum:

```bash
sudo cp zapret/etc/zapret.sh /etc/init.d/zapret
sudo chmod +x /etc/init.d/zapret
sudo rc-update add zapret default
sudo rc-service zapret start
```

Doğrulama:

```bash
rc-service zapret status
ss -ltnp | grep 20090
```

## 4. systemd Servisi (Alternatif)

systemd kullanan dağıtımlar (Arch, Fedora, Debian) için `/etc/systemd/system/zapret.service`:

```ini
[Unit]
Description=zapret SOCKS5 DPI-evasion proxy
After=network.target

[Service]
Type=simple
User=KULLANICI
Group=KULLANICI
ExecStart=/usr/local/bin/tpws --socks --port=20090 --bind-addr=127.0.0.1 \
  --filter-tcp=443 --split-pos=1,midsld --disorder --oab=tls --tlsrec=1 \
  --hostlist-auto=/etc/zapret/auto-hostlist.txt --hostlist-auto-fail-threshold=1
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Kurulum:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now zapret
```

Doğrulama:

```bash
systemctl status zapret
ss -ltnp | grep 20090
```

## 5. İstemci Yapılandırması

Proxy hazır. Şimdi uygulamaların proxy'yi kullanması gerekiyor.

### 5.1. Tarayıcılar — SOCKS5 Profil (Firefox)

Firefox'ta yeni bir profil oluşturun (`firefox -P`), SOCKS5 proxy ayarlayın:

- **Proxy ayarı:** SOCKS v5
- **Sunucu:** `127.0.0.1` (aynı makine) veya `192.168.1.96` (LAN'dan)
- **Port:** `20090`
- **"SOCKS hostları için proxy DNS"** kutusunu **işaretleyin** (kritik — hostlist eşleşmesi için remote DNS şart)

Menü girdisi:

```ini
[Desktop Entry]
Name=Firefox (zapret)
Exec=firefox -P zapret
Icon=firefox
Type=Application
Categories=Network;WebBrowser;
```

### 5.2. Electron Uygulamaları (Discord, Slack, v.b.)

Electron tabanlı uygulamalar `LD_PRELOAD` (tsocks) ile **çalışmaz** — Chromium network stack'i LD_PRELOAD'u bypass eder. Bunun yerine `--proxy-server` flag'i kullanın:

```bash
discord --proxy-server=socks5://127.0.0.1:20090
```

Menü girdisi örneği (`~/Scripts/discord-zapret`):

```bash
#!/bin/bash
exec discord --proxy-server=socks5://127.0.0.1:20090 "$@"
```

```ini
[Desktop Entry]
Name=Discord (zapret)
Exec=~/Scripts/discord-zapret
Icon=discord
Type=Application
Categories=Network;Chat;
```

> **GUI uygulama için `$DISPLAY` şart.** Masaüstünüzden (KDE/GNOME) menü ile açın veya terminalde `export DISPLAY=:0` ile.

### 5.3. Diğer Uygulamalar — tsocks

Electron olmayan uygulamalar (terminal araçları, oyunlar, v.b.) için `tsocks` kullanılabilir:

```bash
sudo emerge tsocks        # Gentoo
sudo apt install tsocks   # Debian/Ubuntu
```

`/etc/tsocks.conf`:

```
server = 127.0.0.1
server_port = 20090
```

Kullanım:

```bash
tsocks some-app
```

> tsocks, `LD_PRELOAD=libtsocks.so` ile `socket()`/`connect()` çağrılarını yakalar. Electron'lar bunu bypass eder; onlar için `--proxy-server` kullanın.

## 6. Doğrulama

### Proxy dinliyor mu?

```bash
ss -ltnp | grep 20090
```

### Bir site proxy üzerinden açılıyor mu?

```bash
curl --socks5-hostname 127.0.0.1:20090 -s -o /dev/null -w "%{http_code}\n" https://example.com
```

### Hostlist otomatik algılama çalışıyor mu?

```bash
cat /etc/zapret/auto-hostlist.txt
```

(Engellenmiş domain'ler otomatik olarak listeye eklenir.)

## 7. Sorun Giderme

| Sorun | Çözüm |
|-------|-------|
| `bind: Permission denied` | Port >1024 seçin (`20090` gibi) veya root olarak çalıştırın |
| Site açılmıyor (timeout/RST) | Sitenin hostlist'ta olduğundan emin olun; otomatik algılama `--hostlist-auto-fail-threshold=1` ile 1 hata sonrası devreye girer |
| Electron'da hata, tsocks'ta çalışıyor | Electron `--proxy-server=socks5://...` ile açın (tsocks'u bypass eder) |
| GUI uygulama açılmıyor | `$DISPLAY` ortam değişkeni dolu olmalı (masaüstünden açın) |
| Diğer siteler yavaş/bozuk | Hostlist dar tutun; sadece engellenmiş domain'ler desync'lenir, geri kalanı normal geçer |
| `segmentation failed` log uyarısı | Desync tam uygulanamadı ama bağlantı başarılı olabilir; sorun devam ederse parametreleri test edin |

## 8. Güvenlik

- Proxy **yalnızca LAN'a veya localhost'a** açılsın (`--bind-addr=127.0.0.1` veya `192.168.1.x`) — `0.0.0.0` sadece güvenilen ağlarda
- Hostlist dosyası (`auto-hostlist.txt`) düzenli kontrol edin; istenmeyen domain'leri silin
- Zapret **yalnızca TCP/443** trafiği desync'ler; HTTP (80) ve diğer portlar geçişli

---

*Kaynak: [github.com/bol-van/zapret](https://github.com/bol-van/zapret) (MIT). Bu rehber Gentoo/OpenRC varsayılan, systemd alternatifi ile hazırlanmıştır.*
