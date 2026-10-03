# Cudy WR3000 v1 · расширение SPI NOR до 128 МиБ

[![OpenWrt](https://img.shields.io/badge/OpenWrt-v24.10.4-00B5E2?logo=openwrt&logoColor=white)](https://github.com/openwrt/openwrt/tree/v24.10.4)
![Устройство](https://img.shields.io/badge/Cudy-WR3000%20v1-34495E)
![Flash](https://img.shields.io/badge/SPI%20NOR-128%20МиБ-2E8B57)

Исходники OpenWrt для **Cudy WR3000 v1** после аппаратной замены штатной
16-МиБ SPI NOR на 128-МиБ микросхему. Проект расширяет раздел `firmware`
в дереве устройств, чтобы OpenWrt видел дополнительное пространство флеш-памяти.

> [!IMPORTANT]
> Эта сборка предназначена для роутера **с уже заменённой 128-МиБ SPI NOR**.
> Она не предназначена для WR3000 v1 со штатной 16-МиБ флеш-памятью и для
> других моделей WR3000. Перед прошивкой проверьте модель устройства, размер
> установленной микросхемы и возможность её адресации загрузчиком.

## Что изменено

Основа — [OpenWrt v24.10.4](https://github.com/openwrt/openwrt/tree/v24.10.4),
коммит `78b23a26c4c98938d549e7ff5876508544e33d4d`. В
`target/linux/mediatek/dts/mt7981b-cudy-wr3000-v1.dts` изменён один
параметр раздела `firmware`:

| | Штатная разметка | Этот проект |
| --- | ---: | ---: |
| Начало раздела | `0x000F0000` | `0x000F0000` |
| Длина раздела | `0x00F10000` | `0x07E10000` |
| Конец раздела | 16 МиБ (`0x01000000`) | 127 МиБ (`0x07F00000`) |

Последний 1 МиБ 128-МиБ микросхемы остаётся за пределами раздела
`firmware`. Второе значение в свойстве DTS `reg` — **длина раздела**,
а не адрес его конца.

Лимит размера **собираемого образа** в `filogic.mk` оставлен штатным:
`IMAGE_SIZE := 15424k`. Расширение раздела и размер файла прошивки — разные
величины.

## Состояние проекта

По сообщению автора, собранная прошивка работает на WR3000 v1 с заменённой
128-МиБ флеш-памятью. Этот репозиторий содержит исходники и конфигурацию
сборки. Аппаратная замена микросхемы и изменения загрузчика в него не входят;
загрузчик должен уметь обращаться к установленной флеш-памяти.

## Как собрать

Понадобятся Linux, [зависимости сборки OpenWrt](https://openwrt.org/docs/guide-developer/toolchain/install-buildsystem)
и около 20 ГиБ свободного места для временных файлов.

```sh
git clone git@github.com:cblp0k/cudy_wr3000-v1_flash_extension.git
cd cudy_wr3000-v1_flash_extension
./scripts/feeds update -a
./scripts/feeds install -a
cp configs/cudy-wr3000-v1.config .config
make defconfig
make -j"$(nproc)" V=s
```

Готовые образы появятся в `bin/targets/mediatek/filogic/`. Для этой
конфигурации ожидаются файлы
`openwrt-mediatek-filogic-cudy_wr3000-v1-initramfs-kernel.bin` и
`openwrt-mediatek-filogic-cudy_wr3000-v1-squashfs-sysupgrade.bin`.

Файл [`configs/cudy-wr3000-v1.config`](configs/cudy-wr3000-v1.config)
содержит сокращённую конфигурацию исходной сборки. Ревизии использованных
тогда feeds записаны в [`configs/feeds.buildinfo`](configs/feeds.buildinfo).
Команда `feeds update -a` получает их текущие версии; для точного повторения
исходной сборки нужно выбрать ревизии из `feeds.buildinfo` до установки
пакетов.

Полный локальный `.config`, ключи подписи и другие личные данные в Git
не добавляются. Параметр `CONFIG_BUSYBOX_DEFAULT_PASSWD=y` в развёрнутой
конфигурации — это булева настройка BusyBox, а не значение пароля. Пароли
устройства задавайте отдельно после установки.

## Проверка на устройстве

После установки на модифицированное устройство можно сверить размер флеш-памяти,
раздел `firmware` и доступное место overlay:

```sh
dmesg | grep -i spi-nor
cat /proc/mtd
df -h /overlay
```

Размер раздела `firmware` по этой DTS-разметке — `0x07E10000` байт
(примерно 126 МиБ). Доступное место overlay зависит от фактического образа
и состояния файловой системы. Порядок установки OpenWrt и способы восстановления
устройства приведены на [странице Cudy WR3000 v1 в OpenWrt Wiki](https://openwrt.org/toh/cudy/wr3000_v1).

## Основа и лицензии

Проект основан на [OpenWrt](https://github.com/openwrt/openwrt).
Оригинальный вводный документ сохранён как
[`README.openwrt.md`](README.openwrt.md). Условия лицензирования исходных
файлов указаны в [`COPYING`](COPYING), `LICENSES/` и заголовках файлов.
