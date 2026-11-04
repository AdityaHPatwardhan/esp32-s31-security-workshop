# ESP32-S31 security workshop: firmware and Launchpad config

## Flash Assignment 1 from your browser

**[Open ESP Launchpad and flash the TLS example](https://espressif.github.io/esp-launchpad/?flashConfigURL=https://raw.githubusercontent.com/AdityaHPatwardhan/esp32-s31-security-workshop/main/launchpad/config.toml)**

```
https://espressif.github.io/esp-launchpad/?flashConfigURL=https://raw.githubusercontent.com/AdityaHPatwardhan/esp32-s31-security-workshop/main/launchpad/config.toml
```

1. Use Chrome or Edge. Plug the board's **USB-to-UART** USB-C port into your laptop.
2. Click **Connect**, pick the serial port, then **Flash**.
3. Open **Console**. On Wi-Fi `EspressifEvent` you should see `esp-x509-crt-bundle: Certificate validated` and `Certificate verified.`

Companion files for the [Introduction to the ESP32-S31: Security](https://developer.espressif.com/workshops/introduction-to-esp32-s31-security/) workshop.

## firmware/https_mbedtls

The stock ESP-IDF `examples/protocols/https_mbedtls` example built for **esp32s31** with **ESP-IDF v6.1**
(Assignment 1 of the workshop). Settings in `sdkconfig.defaults`:

- Wi-Fi SSID `EspressifEvent`, open network (no password)
- Certificate bundle enabled, 4 MB flash, single large factory app partition

| File | Flash offset |
|------|--------------|
| `https_mbedtls-esp32s31-merged.bin` | `0x0` (everything in one image) |
| `bootloader.bin` | `0x2000` |
| `partition-table.bin` | `0x8000` |
| `https_mbedtls.bin` | `0x10000` |

Flash from the command line:

```bash
python -m esptool --chip esp32s31 -p PORT write-flash 0x0 https_mbedtls-esp32s31-merged.bin
```

Or flash from the browser with ESP Launchpad (Chrome/Edge, Web Serial): see `launchpad/`.

Expected log after boot: `example_connect` gets an IP on `EspressifEvent`, then
`esp-x509-crt-bundle: Certificate validated`, `Certificate verified.` and the cipher suite line.

## launchpad/

Browser flashing with [ESP Launchpad](https://espressif.github.io/esp-launchpad/) (Chrome or Edge, USB cable on the board's USB Serial/JTAG port):

- Quick Start flow (app and chip dropdowns):
  <https://espressif.github.io/esp-launchpad/?flashConfigURL=https://raw.githubusercontent.com/AdityaHPatwardhan/esp32-s31-security-workshop/main/launchpad/config.toml>
- Minimal flow (one Connect button, flashes the merged image at 0x0):
  <https://espressif.github.io/esp-launchpad/minimal-launchpad/?flashConfigURL=https://raw.githubusercontent.com/AdityaHPatwardhan/esp32-s31-security-workshop/main/launchpad/minimal.toml>
