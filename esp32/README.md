# ESP32 firmware

Runs the panel from an ESP32 instead of the Raspberry Pi, using the
[HUB75 adapter for ESP32-DevKitC V4 / ESP32-S3 DevKitC-1](https://www.amazon.fr/dp/B0FVGCF1RW)
(seengreat "RGB Matrix Adapter Board (E)"). Built with
[PlatformIO](https://platformio.org/) (`pip install platformio`).

| Environment | Board | Transport | Status |
|---|---|---|---|
| `esp32-classic` | ESP32-DevKitC V4 | Classic Bluetooth (SPP), same as the Pi | **Works end to end** (adapter rev 2.2, 64x32 panel, Pixel 9) |
| `esp32-s3-ble` | ESP32-S3-DevKitC-1 | BLE (the S3 has no classic Bluetooth) | Compiles only - never run on hardware |
| `esp32-test-pattern` | ESP32-DevKitC V4 | none | Static diagnostic pattern - quickest panel wiring/timing check |

## Setup (esp32-classic)

1. With everything unpowered, plug the ESP32 into the adapter and the
   panel's ribbon cable into the adapter's HUB75 port.
2. Power the panel through the adapter's USB-C or DC jack (5V, 4A+). The
   ESP32's own USB only powers the ESP32.
3. Plug the ESP32's USB into the computer and flash:
   ```
   cd esp32
   pio run -e esp32-test-pattern -t upload   # optional: crisp white lines + red/green/blue dots
   pio run -e esp32-classic -t upload
   ```
4. On the phone, pair with **`teslapi-esp32`** in Bluetooth settings.
   Android will then say "Can't connect" - that's normal, it only means
   the ESP32 has no audio profile.
5. In the TeslaLED app, tap the transport button (shows "PI") → **ESP32
   Standard**, and enter the ESP32's Bluetooth address (shown in the
   phone's Bluetooth device details). For this board: `68:09:47:F8:B5:9A`.

## Things to know

- **Pin mapping** (`src/adapter_pins.h`) matches none of the library
  defaults, and differs between adapter revisions V1.x and V2.x - check
  the revision printed on the board
  ([seengreat wiki](https://seengreat.com/wiki/186/rgb-matrix-adapter-board-e)).
- **Clock phase is flipped** (`clkphase = false` in `panel.h`). With the
  library default, white pixels fringed into neighbouring columns (colour
  channels a pixel apart), worst on the bottom half. Solid colours hide
  this; the test pattern's thin white lines show it.
- **The Arduino core is pinned to 3.3.12** in `platformio.ini` (via the
  pioarduino platform). PlatformIO's stock platform ships core 2.0.17,
  which Android 17 can't connect to over Bluetooth. Core 4.0 disables
  `BluetoothSerial` by default, so don't upgrade blindly.
- **RAM is tight.** Bluetooth crashes on connect if it runs short, so:
  Bluetooth starts before the panel, BLE memory is released, the panel
  is single-buffered, and messages are capped at 8KB (the Pi allows
  64KB; a frame is ~6.2KB). Free heap is printed at boot - ~79KB today.
- **Incoming Bluetooth data bypasses `BluetoothSerial`'s 512-byte queue**,
  which silently drops overflow, and goes into an 8KB buffer instead.
- **Serial monitor at 115200** shows boot, connect and error messages.

## Code

- `main_classic.cpp` / `main_ble.cpp` - the two entry points, chosen per
  environment by `build_src_filter` in `platformio.ini`.
- `protocol.h` - wire protocol, mirrors `protocol.py` at the repo root.
- `panel.h` - HUB75 wrapper (mirrors `panel.py`), built on
  [ESP32-HUB75-MatrixPanel-DMA](https://github.com/mrcodetastic/ESP32-HUB75-MatrixPanel-DMA).
- `adapter_pins.h` - adapter rev 2.x pin mapping.
- `main_test_pattern.cpp` - standalone color test, no Bluetooth.

## Not yet verified

- **The S3/BLE path**: the app now has a BLE client (`BleTransport`), but
  it has never been tested against this firmware on a real S3. The
  shared fixes (single-buffered panel, 8KB message cap) apply to it; the
  startup-order and receive-buffer fixes were only made in
  `main_classic.cpp`.
