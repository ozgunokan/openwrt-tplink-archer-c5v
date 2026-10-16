# TP-Link Archer C5v — OpenWrt

EcoNet EN751221 tabanlı TP-Link Archer C5v için OpenWrt portu (kernel 6.18).

> **Sorumluluk reddi:** Bu yazılım "olduğu gibi" sunulur, hiçbir garanti verilmez. Kurulumdan doğabilecek arıza, veri kaybı veya cihaz bozulmasından kullanıcı sorumludur. ISP'nin kiraladığı cihazlara kurulumdan doğan hukuki sorunlar da kullanıcıya aittir.

## Donanım

| | |
|---|---|
| SoC | EcoNet EN751221 (MIPS 34Kc, big-endian, 2 VPE) |
| RAM | 128 MB DDR3 |
| Flash | 128 MB SPI NAND (ESMT F50L1G) |
| 2.4 GHz | MT7603 (`kmod-mt7603`) |
| 5 GHz | MT7612EN (`kmod-mt76x2`) |
| UART | `J1059` (4 pin, 115200 8N1, 3.3V) |

## Özellikler

- Donanımsal NAT hızlandırma (PPE/FOE offload), gigabit'te ~930 Mbit/s
- İkinci CPU çekirdeği aktif (VPE/SMP)
- Çoklu iş parçacığı ve ağ yığını iyileştirmeleri
- NAND flash ve genel kararlılık düzeltmeleri
- WAN VLAN: Türk Telekom ve TT altyapılı ISP'ler için `eth0.35`, Türkcell Superonline için `eth0.2`

---

## Kurulum

Kurulum UART üzerinden yapılır. Önce OpenWrt RAM'den (initramfs) açılır, flash yedeklenir, sonra `sysupgrade` ile kalıcı olarak yazılır. Bootloader (`boot` partition'ı) hiçbir adımda değiştirilmez.

### Gerekenler

- 3.3V USB-TTL adaptörü (5V **kullanmayın**)
- Ethernet kablosu
- Tera Term (XMODEM desteği için)
- Bu depodaki dosyalar:

| Dosya | Boyut | Kontrol | Ne işe yarar |
|---|---|---|---|
| `tclinux.bin` | 5.596.176 bayt | MD5 `8c3bb42cc775951863353db586fece4e` (CRC32 `a9eabc95`) | RAM'de açılan OpenWrt (initramfs) |
| `c5v-mover-556410.bin` | 284 bayt | MD5 `9a0926df9ce82a80a133ab066d09c0eb` | İmajı RAM'de doğru adrese taşıyan küçük program |
| `openwrt-econet-en751221-tplink_archer-c5v-squashfs-sysupgrade.bin` | 13.609.227 bayt | MD5 `c1d6d0df63d39cbcdcdc68690800f152` | Flash'a yazılacak kalıcı imaj |

> `tclinux.bin` ile `c5v-mover-556410.bin` birbirine bağlıdır. Mover, bu `tclinux.bin`'in boyutuna göre hazırlanmıştır. Farklı bir initramfs kullanacaksanız mover da yeniden üretilmelidir (bkz. [Teknik notlar](#teknik-notlar)).

### 1. UART bağlantısı

`J1059` header'ındaki pinleri multimetreyle belirleyin (siyah prob GND'de):

- **GND:** Router kapalıyken metal kalkanla bip veren pin
- **VCC:** Router açıkken sabit 3.3V. **Bağlamayın.**
- **TX:** Açılışın ilk saniyelerinde dalgalanan pin
- **RX:** Kalan pin

Bağlantı: adaptör GND → GND, adaptör RX → router TX, adaptör TX → router RX. Tera Term ayarları: **115200, 8N1, flow control kapalı**.

### 2. Bootloader'ı durdur ve rescue sayfasını aç

Router'ı açın. `Press any key in 1 secs to enter boot command mode.` yazısı çıkınca bir tuşa basın. `bldr>` istemi geldiğinde:

```
bldr> httpd
```

Bilgisayarın Ethernet bağlantısına elle IP verin: `192.168.1.2`, maske `255.255.255.0`. Tarayıcıda `http://192.168.1.1` adresini açın.

### 3. `tclinux.bin`'i RAM'e yükle

Rescue sayfasından `tclinux.bin` dosyasını yükleyin. Dosyanın adı tam olarak `tclinux.bin` olmalıdır.

Konsolda şu çıktı görünür. Hata mesajı **normaldir**: dosyada TP-Link başlığı olmadığı için flash'a yazılmaz, sadece RAM'e iner.

```
rcvdata_size = 5596176
start = 0x80020325
Check data fail, upload failed
Fail to upload firmware!
```

`rcvdata_size` ve `start` değerleri bunlardan farklıysa devam etmeyin.

> ⚠️ Rescue sayfası, geçerli bir TP-Link başlığı taşıyan dosyaları **doğrudan flash'a yazar**. Bu sayfaya `tclinux.bin` dışında bir dosya yüklemeyin.

### 4. Mover'ı çalıştır

```
bldr> xmdm a6000000 11c
```

Tera Term'de **File → Transfer → XMODEM → Send** ile `c5v-mover-556410.bin` dosyasını gönderin. Ardından:

```
bldr> dump a6000000 10
```

Çıktı `40 19 60 00 24 18 ff fe 03 38 c0 24 40 98 60 00` olmalı. Doğruysa:

```
bldr> jump a6000000
bldr> dump a0020200 10
bldr> dump a4000000 10
```

Beklenen çıktılar:

```
a0020200  a9 ea bc 95.00 55 64 10.43 4f 50 59 ...   (CRC, boyut, "COPY")
a4000000  00 80 80 25.00 a0 88 25.00 c0 90 25.00 e0 98 25
```

İlk 4 bayt (`a9eabc95`) `tclinux.bin`'in CRC32 değeriyle aynı olmalıdır.

### 5. OpenWrt'yi RAM'den aç

```
bldr> jump 84000000
```

Açılışta `UBI error: no valid UBI magic found inside mtd2` mesajı görünür. Bu normaldir, flash'ta henüz UBI yoktur.

Initramfs'te DHCP sunucusu çalışmaz. Bilgisayardaki `192.168.1.2` IP'si ile router'a `192.168.1.1` adresinden bağlanılır.

### 6. Flash'ı yedekle (atlamayın)

Bu yedek stok firmware'e dönmenin **tek yoludur**. `radio` partition'ında WiFi kalibrasyon verisi bulunur ve bu veri yeniden üretilemez.

Windows'ta **CMD** açın (PowerShell binary veriyi bozar):

```bat
mkdir "%USERPROFILE%\c5v-yedek"
cd /d "%USERPROFILE%\c5v-yedek"
for %i in (0 1 2 3 4 5 6) do ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL root@192.168.1.1 "nanddump -q -a --bb=padbad /dev/mtd%i" > mtd%i.bin
for %i in (0 1 2 3 4 5 6) do @certutil -hashfile mtd%i.bin MD5 | findstr /v ":"
```

Router'da aynı değerleri alıp karşılaştırın:

```sh
for i in 0 1 2 3 4 5 6; do nanddump -q -a --bb=padbad /dev/mtd$i 2>/dev/null | md5sum; done
```

Yedi değerin hepsi eşleşmeli. Yedekleri ayrıca USB belleğe veya buluta kopyalayın.

### 7. Kalıcı kurulum

Windows'tan imajı gönderin:

```
scp -O -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL openwrt-econet-en751221-tplink_archer-c5v-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/fw.bin
```

`-O` için "unknown option" hatası alırsanız bu seçeneği silin.

Router'da:

```sh
md5sum /tmp/fw.bin          # c1d6d0df63d39cbcdcdc68690800f152 olmalı
sysupgrade -T /tmp/fw.bin   # sadece doğrular, bir şey yazmaz
sysupgrade -n /tmp/fw.bin
```

Yazma 1–3 dakika sürer ve router kendiliğinden yeniden başlar. Bu sırada **fişi çekmeyin**.

### 8. İlk açılış

- Bilgisayarı tekrar otomatik IP'ye alın.
- LuCI: `http://192.168.1.1`, kullanıcı adı `root`, şifre yok. İlk girişte şifre belirleyin.
- WAN'ı ISP'nize göre ayarlayın: Türk Telekom için `eth0.35`, Superonline için `eth0.2`.

---

## Stok firmware'e dönüş

> Bu yöntem henüz test edilmedi.

1. Kurulumun 2–5. adımlarıyla initramfs'i açın.
2. Router'da UBI bağlıysa ayırın:

```sh
ubidetach -p /dev/mtd2 2>/dev/null
```

3. Windows'ta yedek klasöründe **CMD** ile iki partition'ı doğrudan SSH üzerinden yazın. mtd2 96 MB olduğu için RAM'e sığmaz, bu yüzden dosyayı `/tmp`'ye kopyalamak yerine akıtarak yazıyoruz:

```bat
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL root@192.168.1.1 "mtd write - kernel" < mtd1.bin
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL root@192.168.1.1 "mtd write - ubi" < mtd2.bin
```

4. Router'da `reboot` yazın.

Stok firmware'in iki slotu da (ana imaj `0x20000`, yedek imaj `0x3220000`) bu yedeklerin içindedir.

---

## Flash düzeni

| mtd | Ad | Başlangıç | Boyut | Not |
|---|---|---|---|---|
| 0 | boot | `0x0000000` | 128 KB | Bootloader, salt okunur |
| 1 | kernel | `0x0020000` | 4 MB | TP-Link v2 başlığı + LZMA kernel |
| 2 | ubi | `0x0420000` | 96 MB | rootfs + overlay (stok ikinci slot dahil) |
| 3 | romfile | `0x6420000` | 128 KB | Taban MAC `0xF100`'de, salt okunur |
| 4 | config | `0x6440000` | 128 KB | |
| 5 | radio | `0x6460000` | 128 KB | 2.4G EEPROM `0x0`, 5G EEPROM `0x8000`, salt okunur |
| 6 | reserve_area | `0x6480000` | 14.75 MB | Boot flag `0x6560000`'de, salt okunur |

Bootloader açılışta `0x6560000`'daki boot flag'i okur (`0` = ana slot). Kernel'i `0x20000`'daki TP-Link başlığına göre `0x80020000` adresine açar ve oraya atlar.

---

## Teknik notlar

**Neden mover gerekiyor?** Bootloader'ın rescue sayfası yüklenen veriyi multipart başlığıyla birlikte `0x80020000`'a yazar. Bu yüzden imaj `0x80020325` gibi hizasız bir adrese düşer. MIPS'te kod 4 bayta hizalı adreste ve derlendiği adreste çalışmalıdır. Mover imajı `0x84000000`'a kopyalar, cache'i temizler ve kopyanın CRC32'sini `0xA0020200`'e yazar.

**Neden initramfs `0x84000000`'a bağlı?** Büyük bir initramfs `0x80020000`'a bağlanırsa, açıcı kodun bellek alanı (`bss` + heap) kernel'in açıldığı `0x81000000` bölgesinin üstüne taşar ve açılış çöker. Bu yüzden initramfs build'inde `CONFIG_ZBOOT_LOAD_ADDRESS=0x84000000` kullanılır. **Flash'a yazılan kernel `0x80020000`'da kalmalıdır**, çünkü bootloader TP-Link başlığındaki bu adrese yükler. Sysupgrade imajını `0x84000000` ayarıyla derlemeyin.

**Neden `xmdm a6000000`?** `xmdm`, veriyi D-cache'ten RAM'e aktarmadan bırakır. Önbellekli adrese (`0x86000000`) yüklenen kod çalıştırılınca işlemci RAM'deki eski veriyi okur ve çöker. Önbelleksiz KSEG1 adresine (`0xA6000000`) yazmak bu sorunu ortadan kaldırır.

**Mover'ı başka bir initramfs için üretmek:** Mover'da imaja bağlı dört değer vardır: kaynak başlangıcı, kaynak sonu (`start + size`), cache döngüsünün sınırı (`0x84000000 + size`, 32 bayta yuvarlanmış) ve CRC döngüsünün sınırı ile kaydedilen boyut. Bunlar mover'daki `lui`/`ori` çiftleridir (ofsetler `0x14`, `0x24`, `0x4C`, `0x90`, `0xEC`).

## Kendin derlemek

```sh
make menuconfig   # Target: econet / en751221, Profile: TP-Link Archer C5v
make -j"$(nproc)"
```

Initramfs'i RAM'den açmak için ayrıca derlerken `target/linux/econet/en751221/config-6.18` dosyasındaki `CONFIG_ZBOOT_LOAD_ADDRESS` değerini `0x84000000` yapın. Sadece initramfs'i alın, sonra ayarı `0x80020000`'a geri döndürün.

## Teşekkür

Ethernet sürücüsünün temeli Caleb James Delisle'ye aittir ([cjdelisle](https://github.com/cjdelisle)). EN751221 için OpenWrt desteğini başlattığı için teşekkürler.
