# Demo: NVS encryption with an HMAC key in eFuse (ESP32-S31)

ESP-IDF v6.1 example `examples/security/nvs_encryption_hmac`, built for `esp32s31`.
It protects **only the NVS partitions**. No Secure Boot, no flash encryption; the app stays plaintext in flash.

**Binary (merged, flash at 0x0 / load in the emulator):**
[`nvs_encryption_hmac-esp32s31-merged.bin`](https://raw.githubusercontent.com/AdityaHPatwardhan/esp32-s31-security-workshop/main/firmware/nvs_encryption_hmac/nvs_encryption_hmac-esp32s31-merged.bin)

## What happens on the device

1. Boot: `nvs_sec_provider` looks at eFuse **BLOCK_KEY0** (`CONFIG_NVS_SEC_HMAC_EFUSE_KEY_ID=0`).
2. Block empty: it generates a random 32-byte key and **burns it with purpose HMAC_UP. This is permanent.**
   Block already holds an HMAC_UP key: it is reused. Block used for anything else: the app stops with
   "Configured eFuse block for HMAC key already-in-use".
3. The NVS XTS keys are derived from that key through the HMAC peripheral at every boot, so they never sit in flash.
4. The `nvs` and `custom_nvs` partitions are initialised encrypted and a few values are written and read back.

The board stays fully reflashable afterwards; only that one key block is consumed.

## Run it in the browser emulator (nothing is burned on real hardware)

1. Open <https://espressif.github.io/esp-emulator/>.
2. Target → Chip → **ESP32-S31**.
3. Drop `nvs_encryption_hmac-esp32s31-merged.bin` into the firmware box, click **Run**.
4. Click **Reset**, then **Run** again: the key-generation lines are gone, the eFuse survived the reset.
   Reloading the page or loading a file again gives a blank chip.

No network is used by this example.

Native tool, same image:

```bash
esp-emu --chip esp32s31 --firmware nvs_encryption_hmac-esp32s31-merged.bin --efuse lab.efuse --save-state
```

## Flash a real board

Browser: [ESP Launchpad](https://espressif.github.io/esp-launchpad/?flashConfigURL=https://raw.githubusercontent.com/AdityaHPatwardhan/esp32-s31-security-workshop/main/launchpad/config.toml),
pick **nvs_encryption_hmac** in the application dropdown, Connect, Flash.

Command line, from the ESP-IDF shell:

```bash
python -m esptool --chip esp32s31 -p PORT write-flash 0x0 nvs_encryption_hmac-esp32s31-merged.bin
```

## Expected log

First boot:

```
I nvs_sec_provider: NVS Encryption - Registering HMAC-based scheme...
E nvs_sec_provider: Could not find HMAC key in configured eFuse block!
I nvs: Generating NVS encr-keys...
I efuse: BURN BLOCK4
I efuse: BURN BLOCK4 - OK (write block == read block)
I efuse: BURN BLOCK0
I nvs: NVS partition "nvs" is encrypted.
I example: NVS partition "custom_nvs" is encrypted.
I example: Key: u8_key | Val: 255
...
I example: Key: str_key | Val: Lorem ipsum dolor sit amet, consectetur adipiscing elit.
```

(BLOCK4 is the raw eFuse block number of BLOCK_KEY0 on the ESP32-S31; the BLOCK0 write sets its purpose.)

Every later boot:

```
I nvs_sec_provider: NVS Encryption - Registering HMAC-based scheme...
I nvs: NVS partition "nvs" is encrypted.
I example: NVS partition "custom_nvs" is encrypted.
I example: Key: u8_key | Val: 255
```

## Optional: burn your own key from the host

```bash
python $IDF_PATH/components/nvs_flash/nvs_partition_generator/nvs_partition_gen.py \
    generate-key --key_protect_hmac --kp_hmac_keygen --kp_hmac_keyfile hmac_key.bin --outdir keys
python -m espefuse --chip esp32s31 -p PORT burn-key BLOCK_KEY0 keys/hmac_key.bin HMAC_UP
python -m espefuse --chip esp32s31 -p PORT summary | grep KEY_PURPOSE_0
```

The firmware then finds the key and skips generation. Against the native emulator, start it in download mode
(`--strap-mode 0x02 --uart-tcp 127.0.0.1:5555 --efuse lab.efuse --save-state`) and use `-p socket://127.0.0.1:5555`.

## Build it yourself

```bash
cp -r $IDF_PATH/examples/security/nvs_encryption_hmac .
cd nvs_encryption_hmac
printf 'CONFIG_IDF_TARGET="esp32s31"\nCONFIG_NVS_SEC_HMAC_EFUSE_KEY_ID=0\n' >> sdkconfig.defaults
idf.py --preview set-target esp32s31 build
cd build && python -m esptool --chip esp32s31 merge-bin -o nvs_encryption_hmac-esp32s31-merged.bin @flash_args
```
