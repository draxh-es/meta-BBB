# meta-BBB

BeagleBone Black (TI AM335x, Cortex-A8, 32-bit, 512 MB RAM) için AGL Unagi (scarthgap) katmanı.

Kaynak ağacı [beagleboardManifest](https://github.com/draxh-es/beagleboardManifest) ile kurulur. Bu katman tek başına klonlanmaz, manifest onu ağacın köküne indirir.

## Ne yapar

AGL, Unagi'de BeagleBone makine şablonunu kaldırdı (meta-agl `91ef89dc`). Bu katman o şablonu geri getiriyor ve meta-ti'nin yeni `meta-beagle` katmanına göre güncelliyor. Böylece:

```
source meta-agl/scripts/aglsetup.sh -m beaglebone -b build-bbb agl-devel
```

çalışır. Makine ayarlarının geri kalanı (tune, kernel config, wks) AGL ve meta-ti'den gelir. Bu katman şimdilik tarif (recipe) içermiyor.

| Dosya | Açıklama |
|---|---|
| `conf/layer.conf` | Katman kimliği `agl-bbb`, bağımlılıklar `core aglcore meta-beagle`. |
| `templates/machine/beaglebone/20_local.conf.inc` | `MACHINE = "beaglebone"` + `agl_beaglebone.inc` |
| `templates/machine/beaglebone/50_bblayers.conf.inc` | meta-arm, meta-arm-toolchain, meta-ti-bsp, meta-beagle ve bu katman |

Ortaya çıkan yapılandırma (`bitbake -e agl-image-minimal`):

| Değişken | Değer |
|---|---|
| `MACHINE` | `beaglebone` |
| `DEFAULTTUNE` | `armv7athf-neon` |
| `PREFERRED_PROVIDER_virtual/kernel` | `linux-bb.org` (6.12) |
| `PREFERRED_PROVIDER_virtual/bootloader` | `u-boot-bb.org` (2025.x) |
| `WKS_FILE` | `sdimage-bootpart-uuid.wks` (vfat boot + ext4 rootfs) |
| `SERIAL_CONSOLES` | `115200;ttyS0` |

## Derleme

```
cd ~/agl-bbb
source meta-agl/scripts/aglsetup.sh -m beaglebone -b build-bbb agl-devel
```

`conf/local.conf` sonuna (isteğe bağlı, bilgisayara göre):

```
DL_DIR = "/home/<kullanici>/yocto-cache/downloads"
SSTATE_DIR = "/home/<kullanici>/yocto-cache/sstate-bbb"
INHERIT += "rm_work"
```

```
bitbake agl-image-minimal
```

Çıktı: `tmp/deploy/images/beaglebone/agl-image-minimal-beaglebone.rootfs.wic.xz` (+ `.wic.bmap`)

## SD karta yazma

```
lsblk                                   # SD kartın hangi disk olduğunu bulun (ör. /dev/sdb)
sudo bmaptool copy \
  tmp/deploy/images/beaglebone/agl-image-minimal-beaglebone.rootfs.wic.xz /dev/sdX
```

`/dev/sdX` yerine doğru diski yazın. Yanlış disk seçilirse o diskteki her şey silinir.

## Kartı açma

1. **Seri konsol:** J1 header'a 3.3V USB-TTL (FTDI) kablosu bağlayın: GND pin 1, RX pin 4, TX pin 5.
   ```
   picocom -b 115200 /dev/ttyUSB0
   ```
2. **SD'den boot:** BBB varsayılan olarak eMMC'den açılır. SD kart takılıyken **S2 (BOOT) butonunu basılı tutarak** güç verin, U-Boot çıktısı gelince bırakın.
3. Beklenen çıktı: U-Boot SPL → U-Boot → kernel → login satırı:
   ```
   Automotive Grade Linux 21.0.4 beaglebone ttyS0
   beaglebone login: root
   ```
   `agl-devel` ile root şifresizdir.
4. Kontrol: `uname -r` (6.12.x), `cat /proc/device-tree/model`.

## Bilinen durumlar

- AGL'nin CI'ı bu kartı test etmiyor. AGL sürümü güncellenirken BBB build'i elle doğrulanmalı.
- Hedef şimdilik `agl-image-minimal`. Flutter tabanlı demolar 32-bit ARM'da derlenmiyor (AGL'nin şablonu kaldırma gerekçelerinden biri).
- Grafik (weston) sonraki aşama: SGX530 sürücüsü (`ti-sgx-ddk-km`) ve 512 MB RAM sınırı ayrıca ele alınacak.
- AGL kernel config fragment'ları (`linux-agl-config.inc`) kernel 6.12'de "config değeri uygulanmadı" uyarısı verebilir. Gerekirse bu katmana `recipes-kernel/linux/linux-bb.org_%.bbappend` eklenir.
- Kart açılmazsa ilk bakılacak yer boot bölümü: `MLO`, `u-boot.img`, `zImage`, `am335x-boneblack*.dtb` ve `extlinux/extlinux.conf` orada olmalı (`IMAGE_BOOT_FILES`).
