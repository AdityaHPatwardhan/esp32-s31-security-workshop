# Demo: Flash encryption (Development mode) on the ESP32-S31

ESP-IDF v6.1 example `examples/security/flash_encryption` built for `esp32s31` with flash encryption enabled in
**Development mode**. The **bootloader generates the XTS-AES-128 key and burns it on the first boot**, then encrypts
every partition marked `encrypted` in place. NVS encryption rides on top of it (keys in the `nvs_key` partition,
protected by flash encryption). No Secure Boot.

**Binary (merged, flash at 0x0 / load in the emulator):**
[`flash_encryption-esp32s31-merged.bin`](https://raw.githubusercontent.com/AdityaHPatwardhan/esp32-s31-security-workshop/main/firmware/flash_encryption/flash_encryption-esp32s31-merged.bin)

> On a real board the key burn is permanent, JTAG is disabled, and from then on every image written to the flash must
> be encrypted. Development mode leaves one plaintext re-flash (`esptool write-flash --encrypt ...` lets the chip
> encrypt on the way in), so the board is not bricked, but treat it as a committed board.

## What happens on the device

First boot (blank eFuses):

1. `flash_encrypt: Generating new flash encryption key...` → `Writing EFUSE_BLK_KEY0 with purpose 4` (XTS_AES_128_KEY), `BURN BLOCK4`.
2. JTAG and the UART bootloader cache are disabled; in Development mode manual encryption over UART stays enabled.
3. Bootloader, partition table, `storage`, the app, `nvs_key` and `fat_encrypted` are encrypted in place (`Encrypting partition N at offset ...`).
4. `SPI_BOOT_CRYPT_CNT` is burned, `Flash encryption completed`, the chip restarts.

Every later boot: `flash encryption is enabled (1 plaintext flashes left)`, `Flash encryption mode is DEVELOPMENT (not secure)`,
then the app writes 32 bytes to the encrypted `storage` partition, reads them back through the partition API
(plaintext) and through raw `esp_flash_read` (ciphertext), compares an encrypted and a plain FAT partition, and
reports `NVS partition "nvs" is encrypted.`

## Run it in the browser emulator

1. <https://espressif.github.io/esp-emulator/>, Target → Chip → **ESP32-S31**.
2. Drop `flash_encryption-esp32s31-merged.bin` in, **Run**. The first boot takes a while: it encrypts about 1 MB.
3. **Reset**, then **Run** again: no key generation, straight to the status report.

No network is used.

Native tool:

```bash
esp-emu --chip esp32s31 --firmware flash_encryption-esp32s31-merged.bin --efuse lab.efuse --save-state
```

## Expected log (first boot, abridged)

```
I flash_encrypt: Generating new flash encryption key...
I efuse: Writing EFUSE_BLK_KEY0 with purpose 4
W flash_encrypt: Not disabling UART bootloader encryption
I flash_encrypt: Disable UART bootloader cache...
I flash_encrypt: Disable JTAG...
I efuse: BURN BLOCK4
I flash_encrypt: bootloader encrypted successfully
I flash_encrypt: partition table encrypted and loaded successfully
I flash_encrypt: Encrypting partition 2 at offset 0x20000 (length 0x3a410)...
I flash_encrypt: Flash encryption completed
(restart)
I flash_encrypt: flash encryption is enabled (1 plaintext flashes left)
W flash_encrypt: Flash encryption mode is DEVELOPMENT (not secure)
FLASH_CRYPT_CNT eFuse value is 1
Flash encryption feature is enabled in DEVELOPMENT mode
Writing data with esp_partition_write:   00 01 02 03 ...
Reading with esp_partition_read:         00 01 02 03 ...
Reading with esp_flash_read:             <ciphertext>
I nvs: NVS partition "nvs" is encrypted.
```

## Build it yourself

```bash
cp -r $IDF_PATH/examples/security/flash_encryption . && cd flash_encryption
cat >> sdkconfig.defaults <<'CFG'
CONFIG_IDF_TARGET="esp32s31"
CONFIG_SECURE_FLASH_ENC_ENABLED=y
CONFIG_SECURE_FLASH_ENCRYPTION_MODE_DEVELOPMENT=y
CONFIG_NVS_SEC_KEY_PROTECT_USING_FLASH_ENC=y
CFG
idf.py --preview set-target esp32s31 build
cd build && python -m esptool --chip esp32s31 merge-bin -o flash_encryption-esp32s31-merged.bin @flash_args
```

Notes:

- `CONFIG_NVS_SEC_KEY_PROTECT_USING_FLASH_ENC=y` is required on the S31: enabling flash encryption switches NVS
  encryption on, and the default key scheme there is HMAC, which fails the build with
  "Configured eFuse block (CONFIG_NVS_SEC_HMAC_EFUSE_KEY_ID) out of range". This example ships an `nvs_key` partition for the flash-encryption scheme.
- Expanded merge command: `merge-bin --flash-mode dio --flash-freq 80m --flash-size 4MB -o ... 0x2000 bootloader/bootloader.bin 0xd000 partition_table/partition-table.bin 0x20000 flash_encryption.bin`
- Release mode (`CONFIG_SECURE_FLASH_ENCRYPTION_MODE_RELEASE=y`) removes the plaintext re-flash and disables manual
  encryption over UART. That is the production setting and the one the published workshop uses in Assignment 4.
