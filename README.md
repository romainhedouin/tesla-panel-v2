# tesla-panel

Pi-side receiver for the 64×32 RGB LED matrix mounted in the car. Listens
over Bluetooth RFCOMM for commands from the
[TeslaLED](https://github.com/romainhedouin/TeslaLED) Android app and drives
the panel via [rpi-rgb-led-matrix](rpi-rgb-led-matrix/)'s Python bindings.

`rpi-rgb-led-matrix/` is a git submodule pointing straight at
[hzeller/rpi-rgb-led-matrix](https://github.com/hzeller/rpi-rgb-led-matrix)
(no local patches to carry - if that ever changes, fork it first rather
than patching the submodule in place). Clone this repo with `git clone
--recurse-submodules`, or run `git submodule update --init` afterwards if
you already cloned without it - `deploy.sh` needs it checked out locally
to have anything to rsync to the Pi.

`bt_server.py` is the entry point, wiring together `protocol.py` (the wire
protocol - hardware/transport agnostic, covered by `tests/`), `panel.py`
(the RGBMatrix hardware wrapper) and `bt_profile.py` (BlueZ D-Bus service
registration).

`esp32/` runs the panel from an ESP32 instead of the Pi - the classic
Bluetooth version works end to end; see its README for setup.

There is no CI/CD deploying to the Pi itself: `./deploy.sh` copies these files
onto the Pi's home directory (`/home/pi/`) by hand and restarts the systemd
service, but nothing runs that automatically - **a protocol or behavior
change isn't live until you actually run `./deploy.sh`**, and nothing checks
that the Pi is still in sync with what's committed here. `tests/` (`python3
-m pytest tests/`) only exercises `protocol.py`, so it catches wire-format
regressions but nothing hardware- or Bluetooth-related.

## Bill of materials

![Panel back, mounting brackets, and HAT](docs/bill-of-materials.jpg)

- P4 indoor full-color RGB LED matrix module
  - [Reference photo](https://ae-pic-a1.aliexpress-media.com/kf/HTB1wbN2NVXXXXcDXVXXq6xXFXXX2.jpg) of the one used - no longer listed on AliExpress
  - [This listing](https://fr.aliexpress.com/i/32250605175.html?gatewayAdapt=glo2fra) looks similar but is unverified
- Raspberry Pi 3B+
  - A more recent Pi also works, but draws more power - a small fan may become necessary
- [Adafruit HUB75 HAT](https://www.amazon.fr/dp/B00SK69C6E?ref=ppx_yo2ov_dt_b_fed_asin_title)
- [Raspberry Pi 3/3B+ case](https://www.amazon.fr/dp/B09H6KR3JP?ref=ppx_yo2ov_dt_b_fed_asin_title)
  - Had to file off the plastic cover latches to fit the HAT
- [5.5x2.1mm power cable for the HAT](https://www.amazon.fr/dp/B07KFTPF4C?ref=ppx_yo2ov_dt_b_fed_asin_title)
  - Downsides: thin (22 AWG) and long (2m) - thinner/longer cables drop more
    voltage under load, risking under-voltage and visible brightness
    variability. Prefer shorter, thicker cables.
  - If buying today, would try [this one](https://www.amazon.fr/dp/B0FPKWNFLN/ref=sspa_dk_detail_5) instead - still 22 AWG and half the length, which is plenty
- Micro-USB cable for the Pi (USB-C on more recent models)
- M3 screws + washers
- Mounting brackets - the Tesla Model 3's rear window is laterally sloped
  - 2-hole bracket, 2.5cm between the external holes
  - 4-hole bracket, 4.5cm between the external holes
- 2 small carabiners, attached to the mounting brackets
- 2 small suction cups with a loop, attached to the carabiners
- 2-4 magnets, super-glued to the bottom of the Pi case
  - Mount onto the metal sheet below
- 6x16cm metal sheet with holes
  - Unclear original purpose - used here as an anchor point for the magnets
- A soldering iron, to solder the HAT onto the Pi
  - Easy job, just make sure to solder it the right way around or you'll
    have the delightful pleasure of learning to unsolder and re-solder all
    40 tiny little pins... ask how I know...

## Raspberry Pi OS install

Only needed once, on a fresh/replacement SD card, before `pi_side_install.sh`.
Use [Raspberry Pi Imager](https://www.raspberrypi.com/software/):

1. Choose **Raspberry Pi OS Lite (64-bit)** as the OS.
2. Choose the SD card as the storage device - **be very careful you've
   selected the right one**, this erases it.
3. Follow the app's setup prompts (hostname, user, SSH, WiFi, locale) and
   write.

## Wire protocol

`bt_server.py` speaks a length-prefixed binary protocol, matched byte-for-byte
by the Android app's `BluetoothClient`/`AGENTS.md`:

```
[1 byte command type][4 bytes big-endian payload length][payload]
```

followed by a response in the same shape, mirrored - `[1 byte status][4
bytes big-endian message length][message, UTF-8]`. `0x00` is OK (message
normally empty); anything else is an error, with the message carrying the
Pi's own explanation of what went wrong - the Android app surfaces that
text directly (e.g. as an "ERROR: ..." toast) instead of guessing at a
generic failure reason. Command types (kept in sync with the Android app's
constants of the same name):

| Command                 | Value | Payload                                  |
|--------------------------|-------|-------------------------------------------|
| `COMMAND_IMAGE`           | 0     | a 64×32 P6 PPM frame                       |
| `COMMAND_KILL`            | 2     | none - clears the panel                    |
| `COMMAND_SET_BRIGHTNESS`  | 3     | 1 byte, 1-100                              |

RFCOMM is a reliable ordered stream, so the length prefix is all that's
needed to find message boundaries - no sentinel value, no per-chunk acks.

Updates are flicker-free because `panel.py` creates the matrix and its
canvas once at startup and reuses them for the whole process - each command
does an atomic `SwapOnVSync` onto that same canvas, never a process
kill/respawn.

## Bluetooth service registration

`bt_profile.py` registers the RFCOMM service with BlueZ over D-Bus
(`org.bluez.ProfileManager1.RegisterProfile`) instead of PyBluez's
`advertise_service()` - see that file's module docstring for the full
reasoning. Short version: current BlueZ (5.82+, what Debian 13/trixie ships)
removed the legacy `/var/run/sdp` socket interface `advertise_service()`
(and `sdptool`) depend on, so both fail outright on this OS; and it has to
be a real SDP record, not a shortcut around one, because the Android app
connects via `createRfcommSocketToServiceRecord(UUID)`, which looks up the
RFCOMM channel through an actual SDP query at connect time. This also means
BlueZ owns the listening socket - it hands our exported `Profile1` an
already-connected fd per connection, so there's no `listen()`/`accept()`
anywhere in this codebase. Needs `python3-dbus` and `python3-gi` (installed
by `pi_side_install.sh`).

## Deployment

- Three systemd units (`conf/teslabot*.service`), each supervised
  independently via `Restart=always`:
  - `teslabot.service` runs `bt_server.py`.
  - `teslabot-agent.service` keeps a pairing agent
    (`bt-agent -c NoInputNoOutput`) registered.
  - `teslabot-discoverable.service` (a oneshot) powers Bluetooth on and makes
    it discoverable at boot; `/etc/bluetooth/main.conf`'s `DiscoverableTimeout
    = 0` (set by `pi_side_install.sh`) is what makes that stick instead of
    reverting after BlueZ's default 180s.
- Pairing a new phone with the Pi silently fails (no error anywhere, not even
  in `bluetoothctl paired-devices`) unless that pairing agent is running -
  if pairing won't complete, check `systemctl status teslabot-agent` before
  suspecting anything else.
- The Pi's Bluetooth MAC is hardcoded on the Android side
  (`BluetoothClient.findDevice()`); replacing the Pi means updating that
  constant and rebuilding the app.
- On a fresh Pi: `./deploy.sh` from this repo copies it onto the Pi, then
  `ssh teslapi` and run `./pi_side_install.sh` there (packages, compiling
  `rpi-rgb-led-matrix` and its Python bindings, installing the systemd units).
  Safe to re-run - most of its steps are no-ops if already done.
- After that first install, `./deploy.sh` alone is enough for routine code
  changes: it copies the repo over and restarts `teslabot.service`, so the
  new code is live immediately - no manual ssh/restart step needed.

## Troubleshooting

- **Colored static/noise instead of the actual image** is a GPIO-timing or
  wiring issue, not a protocol/software bug - reproduce it with the LED
  matrix library's own test patterns (`rpi-rgb-led-matrix/examples-api-use`)
  with no Bluetooth involved at all before looking anywhere else.
- **Pairing never completes**: see the pairing-agent note above.
- **Panel doesn't update / commands seem to hang**: check `bt_server.log` in
  `/home/pi/` and `journalctl -u teslabot` - a command whose handler raises
  gets logged and answered with an error status rather than silently
  dropped, so a hang usually means the client never got as far as sending
  the header/payload, not a wedged server.
