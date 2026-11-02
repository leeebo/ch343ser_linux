# ch343 linux serial driver
## Description

USB to UART(s) chip CH342/CH343/CH344/CH346/CH347/CH9101/CH9102/CH9103/CH9104/CH9105/CH9111/CH9114/CH9433 are fully compliant to the  Communications Device Class (CDC) standard(Except CH9433), they will work with a standard CDC-ACM driver (CDC - Abstract Control Model). Linux operating systems supply a default CDC-ACM driver that can be used with these USB UART devices. In Linux, this driver file name is cdc-acm.

The CDC-ACM driver has limited capabilities to control specific devices. This generic driver does not have any knowledge about specific device protocols. Because of this, device manufacturers can create an alternate, or custom driver that is capable of accessing the device specific function sets, such as hardware flow control or GPIO functions.

If you use this VCP driver, please check that the CDC-ACM driver was not installed for the USB UART devices mentioned above. You can use command "ls /dev/ttyACM*" to confirm that, to remove the CDC-ACM driver, use command "rmmod cdc-acm".

This driver supports USB to single serial port chip CH343/CH346/CH347/CH9101/CH9102/CH9111/CH9433, USB to dual serial ports chip CH342/CH346/CH347/CH9103, USB to quad serial ports chip CH344, CH9104, CH9105, CH9114, etc.

1. Open "Terminal"
2. Switch to "driver" directory
3. Compile the driver using "make", you will see the module "ch343.ko" if successful
4. Type "sudo make load" or "sudo insmod ch343.ko" to load the driver dynamically
5. Type "sudo make unload" or "sudo rmmod ch343.ko" to unload the driver
6. Type "sudo make install" to make the driver work permanently
7. Type "sudo make uninstall" to remove the driver
8. You can refer to the link below to acquire uart application, you can use gcc or Cross-compile with cross-gcc
   https://github.com/WCHSoftGroup/tty_uart

Before the driver works, you should make sure that the usb device has been plugged in and is working properly, you can use shell command "lsusb" or "dmesg" to confirm that, USB VID of these devices are [1a86], you can view all IDs from the id table which defined in "ch343.c".

If the device works well, the driver will create tty devices named "ttyCH343USBx" in /dev directory, at the same time the driver will create "ch343_iodevx" node for chips that support GPIO signals. Operating the device in the /dev directory under Linux requires root permission by default, if users want to access the device in a non root mode, they can create udev rule file related to the device.

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

## Note

Any question, you can send feedback to mail: tech@wch.cn
