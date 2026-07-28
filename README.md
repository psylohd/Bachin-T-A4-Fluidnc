# Bachin T-A4 Pen Plotter — FluidNC Port



Replacement firmware for the Bachin T-A4 pen plotter using
[FluidNC](https://github.com/bdring/FluidNC) on the stock ESP32-WROOM-32D.
It looks like there is also an older version of the Bachin using a different board? Be aware.

<img width="3472" height="4624" alt="wiredMess" src="https://github.com/user-attachments/assets/f15fe79d-d32d-4f8e-9917-c31c6e756913" />
Note, you don't need to solder any wire (maybe just a jump wire to short GPIO0). Pictured is my board while I was using an external serial adaptor that provided power to the esp32 (before I identified the faulty voltage regulator issue)

## Hardware

| Component | Detail |
|---|---|
| MCU | ESP32-WROOM-32D |
| Board | HW-134A |
| USB-Serial | CH340C (UART0: GPIO1 TX, GPIO3 RX) |
| Stepper drivers | 3× A4988 (X, Y, Z — pen lift) |
| End stops | 2× NO microswitches (X: GPIO35, Y: GPIO34) |

## Pinout

<img width="1440" height="773" alt="image" src="https://github.com/user-attachments/assets/8b7cd1a6-7243-411e-b7eb-9aac2b64c9f8" />

```
              ESP32-WROOM-32D

    GPIO12 ─── X_DIR         X end stop ─── GPIO35
    GPIO14 ─── X_STEP        Y end stop ─── GPIO34
    GPIO26 ─── Y_STEP
    GPIO27 ─── Y_DIR
    GPIO33 ─── Z_STEP        (Z = pen lift stepper)
    GPIO25 ─── Z_DIR

    GPIO1  ─── UART0 TX  ─── CH340C RX
    GPIO3  ─── UART0 RX  ─── CH340C TX
```

End stops are normally-open, active LOW (pulled to GND when pressed).

# Flashing the easy way
GPIO0 (pin 25) needs to be shorted in order to enter the boot mode.

Open & flash using fluidnc webflash https://installer.fluidnc.com/


# Flashing (the hard way)

### Prerequisites
- Python 3 with `platformio` and `esptool`
- ESP32 in boot mode: Short GPIO0 to GND the release.

### Build & flash (WiFi variant)
```bash
git clone https://github.com/bdring/FluidNC
cd FluidNC
cp /path/to/this/config.yaml FluidNC/data/config.yaml
python3 -m platformio run -e wifi -t buildfs
python3 -m platformio run -e wifi

### Erase old NVS (if migrating from stock Grbl_ESP32)
```bash
esptool --chip esp32 --port /dev/ttyUSB?? --baud 921600 \
  --before default_reset --after hard_reset erase_region 0x9000 0x5000
```

Old Grbl_ESP32 NVS settings will crash FluidNC on boot. Erase them first.

# Full flash (first time):
esptool --chip esp32 --port /dev/ttyUSB?? --baud 921600 \
  --before default_reset --after hard_reset write_flash \
  --flash_mode dio --flash_size 4MB --flash_freq 80m \
  0x1000 .pio/build/wifi/bootloader.bin \
  0x8000 .pio/build/wifi/partitions.bin \
  0x10000 .pio/build/wifi/firmware.bin \
  0x3D0000 .pio/build/wifi/littlefs.bin

# Subsequent config updates (filesystem only):
python3 -m platformio run -e wifi -t buildfs
esptool --chip esp32 --port /dev/ttyUSB?? --baud 921600 \
  --before default_reset --after hard_reset write_flash \
  --flash_mode dio --flash_size 4MB --flash_freq 80m \
  0x3D0000 .pio/build/wifi/littlefs.bin
```

After flashing, FluidNC boots a WiFi AP named `FluidNC`. Connect, upload config
via web UI at `http://192.168.0.1/`, or use the filesystem flash method above.

## Usage

| Command | Action |
|---|---|
| `$X` | Unlock from alarm |
| `$H` | Home (Y first to clear clip, then X) |
| `G92 X0 Y0 Z0` | Set current position as origin |
| `G0 X100 Y100` | Jog to position |
| `G0 Z-3` | Pen down |
| `G0 Z0` | Pen up |

## LightBurn Setup

Device: **GRBL**, Origin: **Front Left**, Work area: X=200, Y=300.

Enable **Z Axis Control** with Z Down = -3, Z Up = 0, Z Speed = 3500.

For image engraving (grayscale via variable pen pressure), use the serial proxy:
```bash
python3 serial_proxy.py /dev/ttyUSB20
```
Then point LightBurn at the virtual port it prints. The proxy converts M4/S
laser power commands to Z depth on the fly.

## The Voltage Regulator Incident

The HW-134A uses an AMS1117-3.3 linear regulator to drop
to 3.3V. My powersupply which is really poorly made must had failed because it jumped to 15v and presumably fried the AMS1117-3.3

The AMS1117 was asked to drop 15V → 3.3V, a difference of 11.7V. At ~120mA
that's 1.4 watts of pure heat. My fried regulator spent its short life oscillating
between 0.5V and 5V

A WR1117A-33 pulled from a donor board worked as a replacement :).

Moral: if the label says 12V and your multimeter says 15V, believe the
multimeter.
