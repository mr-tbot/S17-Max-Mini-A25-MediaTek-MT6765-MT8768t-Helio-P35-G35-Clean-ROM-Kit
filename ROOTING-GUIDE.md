# Rooting Guide: MT6765 with Magisk

> **WARNING:** Rooting your device reduces security and may void your warranty.
> This guide is provided for advanced users who understand the risks.
> The main flash kit does NOT root the device.

## Overview

The recommended method for rooting MT6765 devices is **Magisk** — a systemless root solution that:
- Patches the boot image to gain root access
- Does NOT modify the system partition
- Can pass SafetyNet/Play Integrity (with modules)
- Can be fully uninstalled by reflashing the original boot image

## Prerequisites

- Device already flashed with clean GSI (using the main flash script)
- USB debugging enabled
- ADB working on your PC
- A copy of the stock `boot.bin` (included in `backup/stock_partitions/`)

## Method 1: Magisk via mtkclient (Recommended — No TWRP needed)

### Step 1: Get the Boot Image

You need the boot image currently on the device. If you used our flash kit,
the original boot image is preserved (we only flash the super partition).

```bash
# Extract current boot image via mtkclient (device in BROM mode)
cd ~/mtkclient
source venv/bin/activate
python mtk.py r boot boot_for_magisk.bin
```

Or use the backup from the kit:
```bash
cp backup/stock_partitions/boot.bin boot_for_magisk.bin
```

### Step 2: Install Magisk on the Device

1. Download the latest Magisk APK from: https://github.com/topjohnwu/Magisk/releases
2. Transfer to device:
   ```bash
   adb install Magisk-v*.apk
   ```

### Step 3: Patch the Boot Image on the Device

1. Transfer the boot image to the device:
   ```bash
   adb push boot_for_magisk.bin /sdcard/Download/boot.bin
   ```

2. Open the **Magisk** app on the device
3. Tap **"Install"** next to "Magisk" section
4. Select **"Select and Patch a File"**
5. Navigate to `Download/boot.bin` and select it
6. Magisk will create a patched file: `magisk_patched-XXXXX_XXXXX.img`

### Step 4: Pull the Patched Boot Image

```bash
# Find and pull the patched image
adb shell ls /sdcard/Download/magisk_patched*
adb pull /sdcard/Download/magisk_patched-XXXXX_XXXXX.img boot_magisk_patched.bin
```

### Step 5: Flash the Patched Boot Image

```bash
# Put device in BROM mode (power off → Vol Up+Down → plug USB)
cd ~/mtkclient
source venv/bin/activate
python mtk.py w boot boot_magisk_patched.bin
```

### Step 6: Reboot and Verify

1. Unplug USB, power on device
2. Open Magisk app — it should show "Installed" with version number
3. Verify root:
   ```bash
   adb shell su -c id
   # Should output: uid=0(root) gid=0(root)
   ```

## Method 2: Magisk via TWRP (Alternative)

> Note: TWRP for MT6765 is limited. Method 1 is preferred.

If a TWRP build exists for your specific device variant:

1. Flash TWRP to recovery partition via mtkclient:
   ```bash
   python mtk.py w recovery twrp-mt6765.img
   ```
2. Boot into recovery (power off → Vol Up + Power)
3. In TWRP, flash Magisk ZIP from storage

## Unrooting

### Option A: Uninstall from Magisk App
1. Open Magisk → Uninstall → Complete Uninstall

### Option B: Reflash Original Boot Image
```bash
# Put device in BROM mode
python mtk.py w boot backup/stock_partitions/boot.bin
```

## Useful Magisk Modules

| Module | Purpose |
|--------|---------|
| MagiskHide / Shamiko | Hide root from banking apps |
| Universal SafetyNet Fix | Pass Play Integrity |
| Busybox for Android NDK | Extended command-line tools |
| ACC (Advanced Charging Controller) | Battery health management |
| MicroG Installer | Replace Google Services (privacy) |

## Security Considerations

- Root access means any app you grant `su` to has full system access
- Always review root requests carefully in the Magisk app
- Keep Magisk updated to latest version
- Consider enabling "Deny List" for sensitive apps (banking, payments)
- The GPU fix (`debug.renderengine.backend=gles`) will persist — it's in the system init scripts, not the boot image

## Troubleshooting

### Bootloop after flashing patched boot
- Reflash the original boot image:
  ```bash
  python mtk.py w boot backup/stock_partitions/boot.bin
  ```
- Try a different Magisk version

### Magisk app shows "N/A" after reboot
- The patched boot wasn't flashed properly
- Re-patch and re-flash

### SafetyNet/Play Integrity fails
- Install Shamiko module
- Enable "Deny List" and add Google Play Services
- Use Universal SafetyNet Fix module
