# CH343P debug adapter for ESP-Mosaico

This repository documents a CH343P-based driverless USB-UART debug adapter
used with an ESP32-S31 target (ESP-Mosaico core board): the EEPROM
configuration tool `ch343-set-config`, and the test records collected with
the adapter (auto-download, flash write-speed vs. baud rate).

The CH343P enumerates as a standard CDC-ACM port (`/dev/ttyACM*`), so the
stock Linux `cdc_acm` driver is sufficient for everything below. The vendor
VCP driver sources are kept in `driver/` for reference but are not needed.

## Contents

- [USB descriptor configuration (CH343P only)](#usb-descriptor-configuration-ch343p-only)
  - [Which driver to use](#which-driver-to-use)
  - [Usage](#usage)
  - [Notes](#notes)
- [Test records](#test-records)
  - [Environment](#environment)
  - [Auto-download (esptool DTR/RTS reset) test](#auto-download-esptool-dtrrts-reset-test)
  - [Flash write-speed vs. baud rate](#flash-write-speed-vs-baud-rate)

## USB descriptor configuration (CH343P only)

The CH343P has on-chip EEPROM that stores its USB VID, PID, and the
manufacturer, product, and serial strings. `ch343-set-config` in this
repository rewrites those fields in place. Fields that are not passed keep
their current values, and every byte is verified after writing.

### Which driver to use

The script talks to the chip with raw USB control transfers, so the standard
CDC-ACM driver is all you need. No vendor VCP driver, and no
`/dev/ttyCH343USBx` node, is required.

```bash
ls /dev/ttyACM*        # the CH343 enumerates as ttyACM with cdc-acm
```

If you have loaded the vendor `ch343` driver from this repository, unload it
first so the node stays as `ttyACM`:

```bash
sudo rmmod ch343
```

The script needs write access to the USB device node, so run it as root.

### Usage

```
sudo ./ch343-set-config [-V vid] [-P pid] [-m mfr] [-p product] [-s serial]
```

| option | meaning |
| --- | --- |
| `-V`, `--vid` | new Vendor ID, hex |
| `-P`, `--pid` | new Product ID, hex |
| `-m`, `--manufacturer` | manufacturer string, up to 18 chars |
| `-p`, `--product` | product string, up to 18 chars |
| `-s`, `--serial` | serial string, up to 10 chars |

By default the script looks for a device that currently enumerates as
`1a86:55d3`. After changing the IDs, use `--match-vid` / `--match-pid` (or
`--device`) to find it again.

Example — set product and manufacturer strings, keep the default IDs:

```bash
sudo ./ch343-set-config -p "ESP-Mosaico" -m Espressif
```

Changes take effect on the next power-up. Unplug and replug the device, then
verify:

```bash
lsusb -v -d 1a86:55d3 | grep -E 'iManufacturer|iProduct|iSerial'
```

### Notes

- After changing the VID or PID, the vendor VCP driver no longer matches the
  device. It still enumerates as a driverless CDC-ACM port (`ttyACM`).
- Other CH343 variants (CH343G, CH343C, CH343K) do not support EEPROM
  configuration; only the CH343P does.
- A USB VID is registered with USB-IF and owned by its vendor. Only write a
  VID/PID you are authorized to use.
- If the CH343P's RTS pin is pulled low while the chip powers up, the EEPROM
  configuration is ignored and the chip enumerates with the factory strings
  ("USB Single Serial"). Keep the auto-download circuit from loading RTS
  during adapter power-up if you rely on custom strings.

## Test records

### Environment

| item | value |
| --- | --- |
| Adapter | QinHeng CH343P, VID:PID `1a86:55d3`, on `/dev/ttyACM0` (cdc_acm) |
| Target | ESP32-S31 rev v0.0 (ESP-Mosaico core board V0.3), 40 MHz crystal |
| Flash | GD `c8 4018`, 16 MB |
| Wiring | adapter TX/RX -> S31 UART0 RX0/TX0, DTR# -> BOOT, RTS# -> EN, GND; direct connection, no transistor pair |
| Host | Linux 6.x, esptool v5.6.0 (ESP-IDF v6.2-dev) |
| Date | 2026-11-02 |

### Auto-download (esptool DTR/RTS reset) test

esptool's standard reset sequence was driven through the CH343P handshake
lines: DTR# -> BOOT, RTS# -> EN, connected **directly** (the two-transistor
DevKitC circuit is only needed for chips that assert both lines low on port
open, e.g. CH340/CP2102; the CH343 family does not). A 1-10 µF capacitor on
EN to GND is required so BOOT is sampled low when the chip comes out of
reset.

| test | command | result |
| --- | --- | --- |
| Auto-enter download mode | `esptool --chip esp32s31 -p /dev/ttyACM0 chip-id` | PASS — connects on first attempt, chip/MAC read correctly |
| Flash + auto reset back to app | `idf.py -p /dev/ttyACM0 flash` | PASS — write + verify OK, ends with "Hard resetting via RTS pin..." |
| Application output (monitor) | RTS pulse reset, read @115200 | PASS — full ROM + 2nd-stage bootloader + app boot log received |
| eFuse read | `espefuse --chip esp32s31 -p /dev/ttyACM0 summary` | PASS |

Note: the ESP-Mosaico board has a manual on/off power controller (SAM8108).
If esptool reports `Failed to connect ... No serial data received`, first
check the board is actually powered on — plugging in USB does not power the
board up by itself.

### Flash write-speed vs. baud rate

Method: write an 8 MB image of random data to flash offset `0x800000`
(random data defeats esptool compression, so numbers reflect near-raw UART
throughput). Two different images were alternated on consecutive runs so the
diff-flash mechanism could not skip unchanged sectors. Every successful run
ended with `Hash of data verified.` Write time is esptool's own "Wrote ...
in T seconds" figure and includes flash erase; wall clock adds ~12 s for
connect, stub upload, verify and reset.

| baud | runs | result | write time (s) | effective speed (kbit/s) |
| --- | --- | --- | --- | --- |
| 6000000 | 3 | 3/3 PASS | 54.1 / 53.6 / 53.7 | 1240.9 / 1251.2 / 1250.0 |
| 4000000 | 3 | 3/3 PASS | 49.5 / 59.3 / 58.3 | 1355.9 / 1131.8 / 1151.3 |
| 2000000 | 2 | 2/2 PASS | 76.1 / 75.5 | 881.4 / 888.6 |
| 1152000 | 3 | 0/3 FAIL | — | — |
| 460800 | 2 | 2/2 PASS | 197.2 / 197.8 | 340.4 / 339.2 |

Findings:

- **6000000 (the CH343P's maximum rated baud) works reliably** — 3/3 runs
  verified, and it is not faster than 4000000 in practice: at these rates
  flash erase time and protocol overhead dominate (at 4-6 Mbaud only ~15-20 s
  of the ~50-55 s write time is actual UART transfer), so effective
  throughput saturates around 1.2-1.35 Mbit/s no matter how high the baud
  goes. 2000000 is the most robust "fast" setting; 4000000/6000000 are
  equally usable if you want the headline number.
- **1152000 does not work on this link** — deterministic failure right after
  `Changing baud rate to 1152000... Changed.`, the stub then cannot be
  reached (`Unable to verify flash chip connection`). Notably 6000000 — also
  a non-integer divider from the 40 MHz XTAL (6.67, ≈ -0.3 % with the
  fractional divider) — works fine, so a plain crystal-divider mismatch on
  the ESP side does not explain it; the failure is specific to the 1152000
  setting on this CH343P + cdc_acm link. Use 2000000 or 4000000 instead —
  both are faster in practice anyway.
