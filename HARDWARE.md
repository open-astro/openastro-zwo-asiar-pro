# ZWO ASIAIR Pro - Hardware Breakdown

Surveyed live over SSH on a stock ASIAIR Pro (hostname `asiair`, stock ZWO
firmware, kernel 5.15.45-v7l, Raspbian 10 "Buster", 32-bit).

## Board

| Item | Detail |
|------|--------|
| SBC | **Raspberry Pi 4 Model B Rev 1.2** (full 4B, not a compute module) |
| SoC | BCM2711, 4× Cortex-A72 @ 1.5 GHz |
| RAM | 4 GB |
| Storage | microSD (stock: SanDisk 32 GB `SC32G`) - no eMMC |
| Stock OS | 32-bit Raspbian Buster, root mounted **read-only** (`ro` + `noswap` on cmdline) |

Stock SD layout: p1 20 GB FAT32 `/boot` (doubles as image storage), p2 7.5 GB
ext4 rootfs (read-only), p3 240 MB ext4 `/home/pi` (writable settings), p4
2 GB swap (unused).

## Ports

| Port | Backing hardware |
|------|------------------|
| 2× USB 2.0 + 2× USB 3.0 | Pi 4B native (VIA VL817 4-port hub on the USB 2.0 controller, xhci) |
| Gigabit Ethernet | Pi 4B native (`eth0`) |
| 4× 12V DC outputs | Switched by **GPIO 12, 13, 18, 26** - `gpio=18,12,13,26=op,dh,pu` in `/boot/config.txt` (output, default-high, pull-up). Verified live: all four pins OUTPUT/high. This repo bakes the identical line into the OpenAstro image. |
| DSLR shutter ("snap") port | GPIO-driven cable-release (opto-isolated). ZWO's `zwoair_imager` binary references `CableSnap` and a `/dev/pwm-gpio-misc` GPIO/PWM driver; GPIO 19/20/21 are configured as outputs and are the likely snap/status lines. DSLR *data* (image download/control) goes over USB via gPhoto-style control, not this port. |
| microSD slot | Boot + storage |
| HDMI / CSI / DSI | Present on the Pi 4B but unused by ASIAIR firmware |

## Power monitoring

`dtoverlay=i2c3,pins_4_5` puts an extra I2C bus (`/dev/i2c-3`) on GPIO 4/5
for the carrier board's power/current sensing (ZWO ships per-model
`read_power_*.sh` scripts; the aircam variant reads a 12V rail ADC via IIO).
A live scan of i2c-3 found no devices on this unit, so the Pro's sensing is
either the ADC path or absent - the Plus/Mini variants differ.

## Wireless

- **WiFi**: onboard CYW43455, dual-band. Stock firmware runs the ASIAIR
  hotspot on a virtual AP interface **`uap0`** (10.0.0.1/24, hostapd +
  dnsmasq) concurrent with `wlan0` client mode - the exact pattern the
  OpenAstro layer reproduces with NetworkManager on `ap0`.
- **Bluetooth**: onboard, present (ZWO ships `bsa_server`/`bluetooth.sh`) but
  not active in the surveyed state.

## Sound

- **No ALSA device**: no sound cards are registered (`aplay -l`/`arecord -l`
  empty, `/dev/snd` holds only `seq`/`timer`). The Pi 4B's HDMI/headphone
  audio exists in silicon but the ASIAIR firmware doesn't enable it, and the
  case exposes no audio jack.
- **Piezo buzzer on BCM GPIO 19** (GPIO 21 also drives it - verified audibly
  on live hardware with PWM tones on both pins). ZWO's stock firmware has a
  disabled `speaker_beep` routine (pigpio-based) in `zwoair_daemon.sh`. The
  OpenAstro image plays the standard OpenAstro jingle on GPIO 19 at boot
  (`/usr/local/sbin/openastro-beep`, same tune as the ToupTek StellaVita
  image, which has its buzzer on GPIO 12).

## Serial / low-speed buses

- `enable_uart=1`, console on `ttyS0` @ 115200 (`ttyAMA0` also present) -
  useful for headless debug via the GPIO header.
- I2C: only the extra `i2c3` bus (GPIO 4/5); the standard `i2c-1` is not
  enabled. No SPI devices enabled.

## Stock `/boot/config.txt` (effective lines)

```
max_usb_current=1
[pi4]
dtoverlay=vc4-fkms-v3d
max_framebuffers=2
dtoverlay=i2c3,pins_4_5
gpio=18,12,13,26=op,dh,pu
[all]
enable_uart=1
uart_2ndstage=1
```

The one line that matters for OpenAstro is `gpio=18,12,13,26=op,dh,pu` - it
is what turns the four 12V ports on, and `openastro/openastro-setup.sh`
writes the same setting (pin order differs, effect identical).
