# OpenAstro for the ZWO ASIAIR Pro

<img src="https://www.openastro.net/wp-content/uploads/2026/04/OpenAstro_logo.png" alt="OpenAstro logo" width="420">

OpenAstro OS for the **ZWO ASIAIR Pro** (Raspberry Pi 4B 4GB based): a
[Raspberry Pi OS Lite](https://www.raspberrypi.com/software/operating-systems/)
(arm64, no GUI, Debian 13 "Trixie") based image with a WiFi access point and everything ready for
[AlpacaBridge](https://github.com/open-astro/AlpacaBridge).

This repository is identical to
[openastro-raspberrypi](https://github.com/open-astro/openastro-raspberrypi),
with one difference: the ASIAIR Pro's four **12V DC power output ports are
turned on** at boot. The image bakes the following line into
`/boot/firmware/config.txt` (per the
[ASIAIR Pro install doc](https://www.openastro.net/docs/sbc-install/zwo-asiair-pro)):

```
gpio=12,13,26,18=op,dh,pu
```

The OS **runs from the microSD card** - flash it, insert it, power on.

## Supported hardware

| Device | Kernel | Status |
|--------|--------|--------|
| ZWO ASIAIR Pro | Raspberry Pi OS stock | 🚧 In progress |

See [HARDWARE.md](HARDWARE.md) for a full breakdown of the ASIAIR Pro's
internals (board, ports, GPIO map, DSLR snap port, power monitoring),
surveyed live on stock ZWO firmware.

> **ZWO EAF/EFW:** the stock Raspberry Pi OS kernel ships with HIDRAW
> enabled, and the image bakes in a udev rule granting device access, so ZWO
> HID accessories should work out of the box.

## Install

### 1. Download + flash

Grab the latest `openastro-zwo-asiair-pro.img.xz` from the
[latest release](../../releases/latest) page and flash it to a microSD card (8 GB+) with
[Raspberry Pi Imager](https://www.raspberrypi.com/software/),
[balenaEtcher](https://etcher.balena.io/), or `dd`:

```bash
xzcat openastro-zwo-asiair-pro.img.xz | sudo dd of=/dev/sdX bs=4M status=progress conv=fsync
```

(Use Imager's "No customisation" option - credentials and WiFi are already
baked in.)

### 2. Boot

Insert the microSD and power on. The device boots and runs entirely from the
SD card - leave it in. The 12V power output ports come on at power-up.

## First boot defaults

| Setting | Value |
|---------|-------|
| Hostname | `openastro-xxxx` |
| Login | `astro` / `astro` - **change immediately:** `passwd` |
| WiFi AP | `OpenAstro-XXXX` (2.4 GHz, ch 6), password `12345678` |
| AP address | `172.24.1.1` (DHCP for clients) |
| Ethernet | DHCP |
| 12V power ports | On (GPIO 12, 13, 26, 18 driven high at boot) |
| Buzzer | OpenAstro jingle plays on the piezo (GPIO 19) once the system is up (`/usr/local/sbin/openastro-beep`) |

`XXXX` is the last 4 hex digits of the board's WiFi MAC address (e.g.
`OpenAstro-915D`), applied automatically on first boot so multiple boards in
the same place each get a unique hotspot name. The hostname gets the same
suffix in lowercase (`openastro-915d`), so two boards on one home network never
fight over the same DHCP or mDNS name, and AlpacaBridge stamps the same 4
characters on its device names (`915D: ...` in NINA). One ID everywhere.

Reach it over ethernet (`ssh astro@openastro-xxxx.local` or `ssh astro@<ip>`)
or by joining the `OpenAstro-XXXX` WiFi. The access point starts automatically
at every boot, so even if the board can't be reached over your network you can
always join its hotspot and log in at `172.24.1.1`.

### Connect to your own network instead (optional)

All networking is managed by NetworkManager. The hotspot runs on a dedicated
virtual interface (`ap0`), concurrent with `wlan0` client mode, so joining
your own network - from AlpacaBridge's WiFi card in the web portal, or with
`nmcli` (`sudo nmcli dev wifi connect <SSID> password <pass>`) - does **not**
take down the hotspot. (One radio, one channel: while connected as a client
the hotspot follows the client network's channel.) You can also just use the
ethernet port.

## AlpacaBridge

[AlpacaBridge](https://github.com/open-astro/AlpacaBridge) is **preinstalled**
from the OpenAstro apt repository, so the board works at a dark site straight
from the flash - no internet required. When the board does have internet, it
stays current with `sudo apt update && sudo apt upgrade`.

## Build the image yourself

The image is built from a stock Raspberry Pi OS Lite (arm64) image plus the
OpenAstro layer. On an **aarch64** host (an arm64 Debian box, or a Pi itself
- it's a native chroot, no emulation):

```bash
# 1. grab the latest Raspberry Pi OS Lite arm64 image
wget https://downloads.raspberrypi.com/raspios_lite_arm64/images/raspios_lite_arm64-2026-06-19/2026-06-18-raspios-trixie-arm64-lite.img.xz

# 2. bake in the OpenAstro layer and repack
sudo apt install parted e2fsprogs dosfstools
sudo build/build-openastro-image.sh 2026-06-18-raspios-trixie-arm64-lite.img.xz images/openastro-zwo-asiair-pro.img.xz
```

- [`build/build-openastro-image.sh`](build/build-openastro-image.sh) - customizes
  the Raspberry Pi OS image in a chroot and produces a compressed, flashable
  `.img.xz`.
- [`openastro/openastro-setup.sh`](openastro/openastro-setup.sh) - the OpenAstro
  layer (WiFi AP, baked-in credentials, ZWO udev rule, 12V power ports on).
  Idempotent; also runnable directly on a booted board.

## Sibling projects

- [openastro-raspberrypi](https://github.com/open-astro/openastro-raspberrypi)
  - the same OpenAstro layer for stock Raspberry Pi 3B+/4/5 boards.
- [openastro-orangepi4pro](https://github.com/open-astro/openastro-orangepi4pro)
  - same OpenAstro layer for the Orange Pi 4 Pro (Allwinner A733).

## License

See [LICENSE.md](LICENSE.md).
