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
  - [Baud sweep (esptool `chip-id`, link check after baud change)](#baud-sweep-esptool-chip-id-link-check-after-baud-change)
  - [App-level UART test (root-causing the failing baud rates)](#app-level-uart-test-root-causing-the-failing-baud-rates)

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
| 5000000 | 1 | 0/1 FAIL | — | — |
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
- **1152000 and 5000000 do not work on this link** — deterministic failure
  right after `Changing baud rate to ... Changed.`, the stub then cannot be
  reached (`No serial data received` / `Unable to verify flash chip
  connection`). See the sweep below.

### Baud sweep (esptool `chip-id`, link check after baud change)

Each rate was also probed with `esptool -b <baud> chip-id`, which switches
the stub flasher to the requested baud and immediately talks to it — a fast
link-quality check without writing flash. Failing rates were re-run to
confirm determinism.

| baud | result | 40 MHz XTAL divider | ESP-side error |
| --- | --- | --- | --- |
| 460800 | PASS | 86.81 | -0.01 % |
| 921600 | PASS | 43.41 | -0.05 % |
| 1000000 | PASS | 40.0 (exact) | 0 % |
| 1152000 | FAIL (deterministic, 8/8) | 34.72 | +0.10 % |
| 1200000 | PASS | 33.33 | +0.06 % |
| 1500000 | PASS | 26.67 | -0.09 % |
| 2000000 | PASS | 20.0 (exact) | 0 % |
| 3000000 | PASS | 13.33 | +0.16 % |
| 4000000 | PASS | 10.0 (exact) | 0 % |
| 5000000 | FAIL (deterministic, 4/4) | 8.0 (exact) | 0 % |
| 6000000 | PASS | 6.67 | -0.31 % |

An ESP-side crystal-divider mismatch is **ruled out** as the root cause:
5000000 divides the 40 MHz XTAL exactly yet always fails, while 6000000
(-0.31 %) and 3000000 (+0.16 %) pass. The failures are specific to the
1152000 and 5000000 settings on this CH343P (firmware bcdDevice 4.45) +
Linux `cdc_acm` link — i.e. the adapter mis-programs these two rates. The
adapter recovers cleanly: the next connection at any working rate succeeds
immediately after a failure. Avoid 1152000/5000000; use 2000000, 4000000
or 6000000.

### App-level UART test (root-causing the failing baud rates)

To separate "esptool stub problem" from "adapter hardware/firmware
problem", a dedicated test firmware (ESP-IDF app, console disabled, owns
UART0 directly) was flashed to the S31. It cycles through {115200, 1152000,
5000000} baud x {APB, XTAL} clock windows, printing a known counter message
(ESP -> host) and echoing received bytes (host -> ESP). Two extra "scan"
windows sweep the ESP's RX baud across 20 candidate rates (9600-6000000)
while the host floods a probe string at the suspect setting, revealing what
the adapter actually programmed.

| host setting | ESP -> host (prints) | host -> ESP (echo) | scan: actual adapter baud |
| --- | --- | --- | --- |
| 115200 (control) | PASS | PASS 5/5 | — |
| 1152000, clk=APB | FAIL — total silence, 0 bytes | FAIL 0/5 | none of 20 candidates ever framed |
| 1152000, clk=XTAL | FAIL — total silence, 0 bytes | FAIL 0/5 | (same) |
| 5000000, clk=APB/XTAL | FAIL — garbage received | FAIL 0/5 | none of 20 candidates ever framed |

Conclusions:

- **The esptool stub is innocent.** The app fails at exactly the same two
  rates as the stub, even when its UART runs from APB at an essentially
  exact baud (+0.01 %). The ESP32-S31 side — stub or app, XTAL or APB
  clock — is not involved.
- **At the 1152000 setting the CH343P's UART is effectively dead**: the app
  transmits a perfectly good 1152000 signal and the host receives literally
  zero bytes (not even garbage), and nothing the host transmits is ever
  framed by the ESP at any of 20 candidate bauds. Host writes also appear
  to back up while the port is set to 1152000, i.e. the adapter's TX path
  stalls.
- **At the 5000000 setting the CH343P is alive but mis-clocked**: both
  directions produce garbage, and the actual line rate matches none of the
  20 standard candidate bauds.
- This is a CH343P (firmware bcdDevice 4.45) baud-generator bug triggered
  by the specific requested values 1152000 and 5000000. The esptool
  failures at these rates (`No serial data received` right after
  `Changed.`) are fully explained by it.
