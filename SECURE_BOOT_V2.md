# Demo: Secure Boot V2 (RSA-3072) on the ESP32-S31

`hello_world` from ESP-IDF v6.1 with Secure Boot V2 enabled in `sdkconfig.defaults`. Bootloader and app are
signed at build time; the **bootloader burns the key digest and enables Secure Boot on the first boot**.
Nothing else is enabled: no flash encryption, no NVS encryption. Flash stays plaintext.

**Binary (merged, flash at 0x0 / load in the emulator):**
[`secure_boot_v2-esp32s31-merged.bin`](https://raw.githubusercontent.com/AdityaHPatwardhan/esp32-s31-security-workshop/main/firmware/secure_boot_v2/secure_boot_v2-esp32s31-merged.bin)

> **Emulator only.** The signing key `secure_boot_signing_key.pem` is in this public repo. A real board that boots
> this image will trust that public key forever, accept only images signed with it, lose JTAG, and switch ROM
> download mode to Secure mode. Do not flash it to a board you want to keep. It is deliberately not in the Launchpad list.

## What happens on the device

First boot (blank eFuses):

1. Bootloader verifies its own signature and the app signature with RSA-PSS.
2. `Secure boot digests absent, generating..`: the SHA-256 of the public key is burned into **BLOCK_KEY0**
   with purpose `SECURE_BOOT_DIGEST0` (`efuse: BURN BLOCK4`).
3. The two unused digest slots are revoked, `SECURE_BOOT_EN` is set, JTAG is disabled, ROM download mode is
   switched to Secure mode, read-disable of further eFuses is write-protected.
4. `Secure boot permanently enabled`, then the app runs: `Hello world!`

Every later boot: the ROM itself verifies the bootloader (`RSA-PSS secure boot verification succeeded` before
any ESP-IDF log line), the bootloader verifies the app, `secure boot v2 is already enabled, continuing..`.

## Run it in the browser emulator

1. <https://espressif.github.io/esp-emulator/>, Target → Chip → **ESP32-S31**.
2. Drop `secure_boot_v2-esp32s31-merged.bin` in, **Run**. Watch the digest burn and `Secure boot permanently enabled`.
3. **Reset**, then **Run** again: the very first console lines are now the ROM's own verification.

No network is used.

Native tool:

```bash
esp-emu --chip esp32s31 --firmware secure_boot_v2-esp32s31-merged.bin --efuse lab.efuse --save-state
```

With `--save-state` the second run reuses the burned digest. To see Secure Boot reject a tampered app, flip one
byte inside the app region of a copy of the flash image after the first boot and run again: the bootloader
reports a signature failure and refuses to boot.

## Expected log (first boot, abridged)

```
I secure_boot_v2: Secure boot V2 is not enabled yet and eFuse digest keys are not set
I secure_boot_v2: Verifying with RSA-PSS...
I secure_boot_v2: Signature verified successfully!
I secure_boot_v2: Secure boot digests absent, generating..
I secure_boot_v2: Burning public key hash to eFuse
I secure_boot_v2: Application key(0) matches with bootloader key(0).
I secure_boot_v2: Revoking empty key digest slot (1)...
I secure_boot_v2: Revoking empty key digest slot (2)...
I secure_boot_v2: blowing secure boot efuse...
I secure_boot: Enabling Security download mode...
I secure_boot: Disable hardware & software JTAG...
I efuse: BURN BLOCK4
I efuse: BURN BLOCK0
I secure_boot_v2: Secure boot permanently enabled
Hello world!
```

Second boot starts with:

```
Valid secure boot key blocks: 0
RSA-PSS secure boot verification succeeded
```

## Build it yourself

```bash
cp -r $IDF_PATH/examples/get-started/hello_world secure_boot_v2 && cd secure_boot_v2
cat >> sdkconfig.defaults <<'CFG'
CONFIG_IDF_TARGET="esp32s31"
CONFIG_ESPTOOLPY_FLASHSIZE_4MB=y
CONFIG_PARTITION_TABLE_OFFSET=0x10000
CONFIG_SECURE_BOOT=y
CONFIG_SECURE_BOOT_V2_ENABLED=y
CONFIG_SECURE_SIGNED_APPS_RSA_SCHEME=y
CONFIG_SECURE_BOOT_BUILD_SIGNED_BINARIES=y
CONFIG_SECURE_BOOT_SIGNING_KEY="secure_boot_signing_key.pem"
CONFIG_SECURE_BOOT_FLASH_BOOTLOADER_DEFAULT=y
CFG
python -m espsecure generate-signing-key --version 2 --scheme rsa3072 secure_boot_signing_key.pem
idf.py --preview set-target esp32s31 build
cd build && python -m esptool --chip esp32s31 merge-bin --flash-size 4MB -o secure_boot_v2-esp32s31-merged.bin @flash_args
```

Notes:

- `CONFIG_PARTITION_TABLE_OFFSET=0x10000`: the signed bootloader (36 KB) no longer fits below the default 0x8000.
- `CONFIG_SECURE_BOOT_FLASH_BOOTLOADER_DEFAULT=y` puts the bootloader back into `flash_args`. Secure Boot builds
  leave it out by default so that `idf.py flash` cannot overwrite a bootloader by accident; without this option
  the merged image has no bootloader and the chip boots nothing.
- `--flash-size 4MB` is needed because this `flash_args` says `keep`.
- Expanded: `merge-bin --flash-mode dio --flash-freq 80m --flash-size 4MB -o ... 0x2000 bootloader/bootloader.bin 0x10000 partition_table/partition-table.bin 0x20000 hello_world.bin`
