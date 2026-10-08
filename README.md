**en** | [ru](README.ru.md)

# Cudy WR3000 v1 · SPI NOR expansion to 128 MiB

[![OpenWrt](https://img.shields.io/badge/OpenWrt-v24.10.4-00B5E2?logo=openwrt&logoColor=white)](https://github.com/openwrt/openwrt/tree/v24.10.4)
![Device](https://img.shields.io/badge/Cudy-WR3000%20v1-34495E)
![Flash](https://img.shields.io/badge/SPI%20NOR-128%20MiB-2E8B57)

OpenWrt sources for **Cudy WR3000 v1** after replacing the stock
16 MiB SPI NOR chip with a 128 MiB chip. The project expands the `firmware` partition
in the device tree so that OpenWrt can access the additional flash space.

> [!IMPORTANT]
> This build is intended for a router **already upgraded to 128 MiB SPI NOR**.
> It is not intended for a WR3000 v1 with the stock 16 MiB flash or for
> other WR3000 models. Before flashing, check the device model, the capacity
> of the installed chip, and whether the bootloader can address it.

## Flash memory hardware upgrade

On the author's device, the stock **XMC XM25QH128C** was replaced with
a **Winbond W25Q01JVZEIQ**. The SPI NOR capacity increased **eightfold**:
from 16 MiB to 128 MiB.

| Parameter | Stock memory | Installed replacement |
| --- | --- | --- |
| Manufacturer | XMC | Winbond |
| Chip | XM25QH128C | W25Q01JVZEIQ |
| Capacity | 128 Mbit / 16 MiB | 1 Gbit / 128 MiB |
| Package | SOP-8, 208 mil | WSON-8, 8 × 6 mm, code `ZE` |
| Contacts | 8 protruding leads | 8 pads on the underside |
| Contact pitch | 1.27 mm | 1.27 mm |
| Supply voltage range | 2.3–3.6 V | 2.7–3.6 V |

The original model and capacity are confirmed by a saved Linux log:
`spi-nor spi0.0: XM25QH128C (16384 Kbytes)`. After replacement, U-Boot reports
`SF: Detected w25q01jv with page size 256 Bytes, erase size 4 KiB, total 128 MiB`.
The full part number of the installed Winbond chip was confirmed by the person who performed the upgrade.
The stock XMC package was identified from a [WR3000 v1 board photo in the
certification documents](https://fccid.io/2APRGRT02/Internal-Photos/Internal-photos-6580975):
the Linux log does not show the package suffix.

The packages are **different**: SOP-8 has protruding leads, while WSON-8 has contact
pads on the underside. This did not prevent a successful
hardware upgrade on the author's device. The eight-contact variants discussed here
have matching contact pitch and SPI/Quad SPI signal assignments:
`1 — /CS`, `2 — DO/IO1`, `3 — IO2`, `4 — GND`, `5 — DI/IO0`,
`6 — CLK`, `7 — IO3`, `8 — VCC`; their supply voltage ranges also overlap.
However, matching signals and pitch do not mean identical PCB footprints:
whether the chip can be mounted depends on the pads and available space on the specific
board. This upgrade does not guarantee that
SOP-8 and WSON-8 are interchangeable on all devices.

On the Winbond `IQ` variant, the Quad Enable (QE) bit is fixed at one, and the
`/HOLD` function is disabled. Matching SPI signals do not mean that
all additional pin functions are identical. To use capacities above
16 MiB, the bootloader and driver must support addressing the installed
chip; the OpenWrt partition expansion is described below.

The specifications were checked against the **XM25QH128C Rev. 2.1** datasheet
(April 25, 2023, pinout — p. 8, SOP-8 package — p. 97) and
**W25Q01JV Rev. B1** (November 13, 2019, pinout — p. 7,
WSON-8 package — p. 87, full part number — p. 91). Additional sources:

- [XM25QH128C specifications on the XMC website](https://www.xmcwh.com/en/site/product_con/202).
- [Official Winbond 2025 catalog, p. 23](https://www.winbond.com/export/sites/winbond/product-selection-guide/file/2025-Product-Selection-Guide-Winbond-Code-Storage-Flash-Memory.pdf#page=27):
  part number `W25Q01JVZEIQ`, WSON-8 8 × 6 mm package, and 1 Gbit capacity.
- [Winbond W25Q01JV Rev. E datasheet](https://www.mouser.com/datasheet/2/949/Winbond_Electronics_Corporation_09_06_2024_W25Q01J-3501286.pdf):
  p. 5 — pinout, p. 81 — package, pp. 84–85 — ordering and marking.
  The `W25Q01JVZEIQ` package uses the abbreviated marking `25Q01JVIQ`.

## Changes

The change was tested on **OpenWrt 24.10.4** for **Cudy WR3000 v1** after
replacing the flash memory with a 128 MiB Winbond W25Q01JVZEIQ.

You can try applying it to any other OpenWrt version that supports
this router. Before building, check the partition layout and support for
the new memory. The change has not yet been tested on other versions.

Based on [OpenWrt v24.10.4](https://github.com/openwrt/openwrt/tree/v24.10.4),
commit `78b23a26c4c98938d549e7ff5876508544e33d4d`. In
`target/linux/mediatek/dts/mt7981b-cudy-wr3000-v1.dts`, one
parameter of the `firmware` partition was changed:

| | Stock layout | This project |
| --- | ---: | ---: |
| Partition start | `0x000F0000` | `0x000F0000` |
| Partition length | `0x00F10000` | `0x07E10000` |
| Partition end | 16 MiB (`0x01000000`) | 127 MiB (`0x07F00000`) |

The last 1 MiB of the 128 MiB chip remains outside the
`firmware` partition. The second value in the DTS `reg` property is the **partition length**,
not its end address.

The size limit for the **built image** in `filogic.mk` remains at its stock value:
`IMAGE_SIZE := 15424k`. Partition expansion and firmware file size are different
quantities.

## How to build

You will need Linux, the [OpenWrt build dependencies](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem),
and about 20 GiB of free space for temporary files.

```sh
git clone git@github.com:cblp0k/cudy_wr3000-v1_flash_extension.git
cd cudy_wr3000-v1_flash_extension
./scripts/feeds update -a
./scripts/feeds install -a
cp configs/cudy-wr3000-v1.config .config
make defconfig
make -j"$(nproc)" V=s
```

The built images will appear in `bin/targets/mediatek/filogic/`. This
configuration is expected to produce
`openwrt-mediatek-filogic-cudy_wr3000-v1-initramfs-kernel.bin` and
`openwrt-mediatek-filogic-cudy_wr3000-v1-squashfs-sysupgrade.bin`.

The [`configs/cudy-wr3000-v1.config`](configs/cudy-wr3000-v1.config) file
contains the reduced configuration of the original build. The feed revisions used
at that time are recorded in [`configs/feeds.buildinfo`](configs/feeds.buildinfo).
The `feeds update -a` command retrieves their current versions; to reproduce
the original build exactly, select the revisions from `feeds.buildinfo` before installing
packages.

The full local `.config`, signing keys, and other personal data are not added to
Git. The `CONFIG_BUSYBOX_DEFAULT_PASSWD=y` option in the expanded
configuration is a boolean BusyBox setting, not a password value. Set the
device passwords separately after installation.

## Verification on the device

After installation on the modified device, you can check the flash capacity,
the `firmware` partition, and the available overlay space:

```sh
dmesg | grep -i spi-nor
cat /proc/mtd
df -h /overlay
```

The `firmware` partition size in this DTS layout is `0x07E10000` bytes
(approximately 126 MiB). Available overlay space depends on the actual image
and the state of the filesystem. OpenWrt installation instructions and device recovery
methods are provided on the [Cudy WR3000 v1 page in the OpenWrt Wiki](https://openwrt.org/toh/cudy/wr3000_v1).

## Upstream project and licenses

The project is based on [OpenWrt](https://github.com/openwrt/openwrt).
The original introductory document is preserved as
[`README.openwrt.md`](README.openwrt.md). The licensing terms for the source
files are listed in [`COPYING`](COPYING), `LICENSES/`, and the file headers.
