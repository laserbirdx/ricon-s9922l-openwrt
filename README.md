![OpenWrt on Ricon / Alotcer S9922L](assets/banner.webp)

# OpenWrt for Ricon / Alotcer S9922L

OpenWrt 23.05 port for the S9922L industrial 4G router (MediaTek MT7620A, built-in Yuga
CLM920_NC5 LTE modem). The same hardware is sold as **Ricon S9922** and **Alotcer S9922L**.

Tested on a real unit: boots from flash, LAN, Wi-Fi, LuCI, LTE data over QMI and SMS.

## Check your unit first

Before flashing, make sure the stock firmware shows all of these on the console:

- U-Boot banner `U-Boot 1.1.3 (Ralink / Alotcer)`, prompt `Alotcer #`
- stock kernel Linux 2.6.36
- `/proc/mtd`: `uboot`, `factory` at `0x30000`, `config`, `linux` at `0x50000`

Prebuilt images are on the [Releases](../../releases/latest) page.

> **Unofficial.** This board is not in upstream OpenWrt. Flashing overwrites the stock
> firmware. Take a full flash backup first (see below). You do this at your own risk.

## Hardware

| Component | Details |
|---|---|
| SoC | MediaTek MT7620A, MIPS 24Kc @ 580 MHz |
| RAM | 128 MiB DDR2 |
| Flash | 16 MiB SPI-NOR, Winbond W25Q128BV |
| Wi-Fi | MT7620 built-in, 2.4 GHz 802.11n |
| Ethernet | 2 ports on the MT7620 internal switch |
| LTE modem | Yuga CLM920_NC5, USB ID `05c6:9625`, QMI (`/dev/cdc-wdm0`, `wwan0`), AT port `/dev/ttyUSB1` |
| Watchdog | external hardware watchdog fed through GPIO 23 |
| Console | RJ45 console port on the case, 115200 8N1 |
| Stock firmware | Linux 2.6.36, U-Boot 1.1.3 |

### External watchdog

The board has a hardware watchdog chip that resets the router unless GPIO 23 toggles
continuously, so OpenWrt has to feed it from the very start of boot.
[`openwrt/301-mt7620-alotcer-watchdog.patch`](openwrt/301-mt7620-alotcer-watchdog.patch)
toggles GPIO 23 from a kernel timer every 200 ms, starting at `arch_initcall`, and is
required.

### Flash layout

| Offset | Size | Name | Notes |
|---|---|---|---|
| `0x000000` | 192 KiB | `u-boot` | read-only |
| `0x030000` | 64 KiB | `factory` | Wi-Fi EEPROM, MAC address at `0x1028` |
| `0x040000` | 64 KiB | `config` | stock NVRAM, read-only |
| `0x050000` | 15.7 MiB | `firmware` | OpenWrt kernel + rootfs |

The stock U-Boot, `factory` and `config` are kept. Everything after `0x50000` belongs to
OpenWrt.

## Default configuration of the release image

Standard OpenWrt defaults, nothing custom:

| Setting | Value |
|---|---|
| LAN | both Ethernet ports, `192.168.1.1`, DHCP server |
| WAN | **not configured**. See [LTE setup](#lte-setup) |
| Login | `root`, no password |
| Wi-Fi | disabled |
| Web UI | LuCI, with `luci-proto-qmi` |
| Included | `uqmi`, `kmod-usb-net-qmi-wwan`, `kmod-usb-serial-option` |

Set a root password on first login.

## Console

There is no internal UART header to solder. The console is on the **RJ45 console port**
on the case. Use a standard Cisco-style RJ45-to-USB console cable at **115200 8N1**.

- Stock firmware login: `admin` / `admin`
- U-Boot: press **Ctrl+C** when `Press 'Ctrl+c' key to stop boot` appears. The prompt is
  `Alotcer #`.

## Installation

You need a Cisco-style console cable, an Ethernet cable and a TFTP server on your PC.

### 1. Boot the initramfs image from RAM

Put `openwrt-ramips-mt7620-alotcer_s9922l-initramfs-kernel.bin` in the TFTP server root and
give the PC the address `192.168.1.102/24`. Connect the PC to a LAN port.

Power-cycle the router, press Ctrl+C at the U-Boot countdown, then:

```
setenv ipaddr 192.168.1.99
setenv serverip 192.168.1.102
tftpboot 0x81000000 openwrt-ramips-mt7620-alotcer_s9922l-initramfs-kernel.bin
bootm 0x81000000
```

Nothing is written to flash. A power cycle brings the stock firmware back.

### 2. Back up the flash

OpenWrt is now running from RAM at `192.168.1.1`. Save every partition. `factory` holds the
Wi-Fi calibration and MAC address of your unit.

```sh
# on the PC
for i in 0 1 2 3; do
  ssh root@192.168.1.1 "cat /dev/mtd$i" > mtd$i.bin
done
ssh root@192.168.1.1 cat /proc/mtd > proc_mtd.txt
```

`mtd3` (`firmware`) here is the complete stock firmware. Keep it if you want to go back.

### 3. Install to flash

```sh
scp -O openwrt-ramips-mt7620-alotcer_s9922l-squashfs-sysupgrade.bin root@192.168.1.1:/tmp/
ssh root@192.168.1.1 sysupgrade -n /tmp/openwrt-ramips-mt7620-alotcer_s9922l-squashfs-sysupgrade.bin
```

The router reboots into OpenWrt from flash.

### Going back to stock (untested)

Write the `firmware` backup from step 2 back from OpenWrt:

```sh
scp -O mtd3.bin root@192.168.1.1:/tmp/
ssh root@192.168.1.1 "mtd -r write /tmp/mtd3.bin firmware"
```

## LTE setup

The image does **not** include any SIM or WAN settings. Set up the modem after installing.
The modem is a QMI device. LuCI: *Network → Interfaces → Add new interface*, protocol
*QMI Cellular*, device `/dev/cdc-wdm0`, firewall zone `wan`.

From the command line, replacing `internet` with your operator's APN:

```sh
uci set network.wan=interface
uci set network.wan.proto='qmi'
uci set network.wan.device='/dev/cdc-wdm0'
uci set network.wan.apn='internet'
uci set network.wan.pdptype='ipv4'
uci commit network
ifup wan
```

Leave the authentication setting empty. `auth=none` made the modem refuse the data call
(`CALL_FAILED`) on the test unit.

For signal info and SMS in LuCI, `luci-app-3ginfo-lite` and `luci-app-sms-tool-js` (by
4IceG) work with AT port `/dev/ttyUSB1`.

### Package feeds

The image is a 23.05 branch build. If `opkg update` fails because the snapshot feeds are
gone, point `/etc/opkg/distfeeds.conf` at the `23.05.6` release feeds. Userspace packages
install fine from there. Kernel modules will not match this kernel.

## Building from source

The release was built from the OpenWrt `openwrt-23.05` branch at `v23.05.6-30-g33063b4ccf`,
kernel 5.15.211.

1. Copy [`mt7620a_alotcer_s9922l.dts`](mt7620a_alotcer_s9922l.dts) to
   `target/linux/ramips/dts/`.
2. Copy [`openwrt/301-mt7620-alotcer-watchdog.patch`](openwrt/301-mt7620-alotcer-watchdog.patch)
   to `target/linux/ramips/patches-5.15/`.
3. Apply [`openwrt/ramips-mt7620.diff`](openwrt/ramips-mt7620.diff): image recipe
   (`mt7620.mk`), switch and MAC setup (`02_network`), and
   `CONFIG_GPIO_WATCHDOG_ARCH_INITCALL=y` in `config-5.15`.
4. `make menuconfig`: Target `MediaTek Ralink MIPS`, subtarget `MT7620 based boards`,
   profile `Alotcer S9922L`. Then `make -j$(nproc)`.

## Repository layout

| Path | Contents |
|---|---|
| `mt7620a_alotcer_s9922l.dts` | device tree |
| `openwrt/301-mt7620-alotcer-watchdog.patch` | external watchdog feeder (required) |
| `openwrt/ramips-mt7620.diff` | changes to the OpenWrt tree |

## License

[GPL-2.0](LICENSE), the same as OpenWrt and the Linux kernel. The device tree is
dual-licensed `GPL-2.0-or-later OR MIT`, as stated in its header.
