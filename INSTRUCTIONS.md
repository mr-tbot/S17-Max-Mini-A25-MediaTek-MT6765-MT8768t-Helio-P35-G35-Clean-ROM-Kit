# MT6765 De-China: Detailed Build Log & Instructions

## Overview

This document describes exactly how a malware-infected MediaTek MT6765 device was cleaned and flashed with a legitimate Android 12 GSI, including all troubleshooting steps and the critical GPU fix.

**Date:** March 2-3, 2026  
**Device:** MediaTek MT6765/MT8768t, PowerVR GE8320 GPU, ~2.9GB RAM, eMMC  
**Target ROM:** Google GSI `gsi_gms_arm64` Android 12 (SQ3A.220705.003.A1)

---

## Phase 1: Preparation

### 1.1 Tools Required

- **Linux PC** (Ubuntu tested) with USB port
- **Python 3** with pip
- **mtkclient** — MediaTek BROM flashing tool
  ```bash
  git clone https://github.com/bkerler/mtkclient
  cd mtkclient
  python -m venv venv
  source venv/bin/activate
  pip install -r requirements.txt
  ```
- **Android SDK Platform Tools** (for `adb`)
  ```bash
  # Download from https://developer.android.com/tools/releases/platform-tools
  # Or: sudo apt install android-tools-adb
  ```
- **USB cable** (data-capable, short cable preferred for stability)

### 1.2 USB Permissions (Linux)

mtkclient needs raw USB access. Add udev rules:

```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d", MODE="0666"' | sudo tee /etc/udev/rules.d/51-mediatek.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### 1.3 Download the GSI

```bash
mkdir -p ~/gsi && cd ~/gsi
# Google GSI for arm64 with GMS (Google Mobile Services)
# Download from: https://developer.android.com/topic/generic-system-image/releases
# File: gsi_gms_arm64-img-SQ3A.220705.003.A1.zip
unzip gsi_gms_arm64.zip
# This produces: system.img and vbmeta.img
```

---

## Phase 2: Device Backup

### 2.1 Enter BROM Mode

1. Power off the device completely
2. Hold **Volume Up + Volume Down** simultaneously
3. While holding both buttons, plug in the USB cable
4. Device should appear as `ID 0e8d:0003 MediaTek Inc.` in `lsusb`

### 2.2 Full Partition Backup

```bash
cd ~/mtkclient
source venv/bin/activate

# Read ALL partitions (takes ~30 minutes for complete backup)
python mtk.py rl ~/a25_backup --skip userdata

# Critical partitions to verify:
# - super.bin (6.7GB) — contains system, product, vendor
# - boot.bin (32MB) — kernel + ramdisk
# - vbmeta.bin — verified boot metadata
# - lk.bin, lk2.bin — bootloaders
# - recovery.bin — recovery partition
```

> **IMPORTANT:** Always back up BEFORE making any changes. This backup allows full restore.

---

## Phase 3: Unlock Bootloader & Disable Verified Boot

### 3.1 Unlock Bootloader

```bash
# With device in BROM mode:
python mtk.py da seccfg unlock
```

### 3.2 Disable Verified Boot (vbmeta)

Android Verified Boot will reject our GSI since it's not signed by the OEM. We disable it by flashing a vbmeta image with verification flags cleared:

```bash
# The vbmeta.img from the GSI zip has verification disabled
python mtk.py w vbmeta ~/gsi/vbmeta.img
```

The `vbmeta.img` is a 4KB image with the `--flag 2` (disable verification) and `--flag 3` (disable hashtree) flags set.

---

## Phase 4: Build the Modified Super Image

### 4.1 Why We Can't Just Flash system.img

The super partition uses Android's **Logical Partitions (LP)** format. It contains metadata that maps logical partitions (system, vendor, product) to physical offsets. You can't just `dd` a system.img onto the super partition — the LP metadata must be correct.

### 4.2 The GPU Problem

The PowerVR GE8320 GPU in MT6765 devices crashes when Android 12's SurfaceFlinger uses the **Skia GL** rendering backend (default). The fix is to force the **GLES** backend via a system property:

```
debug.renderengine.backend=gles
```

### 4.3 Create GPU Fix Init Script

We inject an init `.rc` script into the system image that sets the GPU property at boot:

```bash
# File: /system/etc/init/gpu-fix.rc
# Contents:
on early-init
    setprop debug.renderengine.backend gles

on boot
    setprop debug.renderengine.backend gles
```

### 4.4 Inject the Script into system.img

```bash
# Using debugfs to inject the file into the ext4 image with correct SELinux context
debugfs -w ~/gsi/system.img << 'EOF'
cd /system/etc/init
write /tmp/gpu-fix.rc gpu-fix.rc
set_inode_field gpu-fix.rc mode 0100644
ea_set gpu-fix.rc security.selinux "u:object_r:system_file:s0"
EOF
```

> **Note:** The system.img uses ext4 with `shared_blocks` feature. The `debugfs` approach works for adding small files but the image cannot be mounted read-write normally.

### 4.5 Build the Super Image

```bash
# The super partition is:
#   - 1MB LP metadata header
#   - system image at offset 0x100000 (1MB)
#   - Padded to exactly 7,134,339,072 bytes total

SUPER_SIZE=7134339072
SYSTEM_SIZE=$(stat -c%s ~/gsi/system_modified.img)

# Create LP metadata + system image
# Using lpmake or manual construction:

# 1. Extract original LP metadata (first 1MB)
dd if=/dev/zero bs=1M count=1 of=~/gsi/super_header.bin

# 2. Use lpmake to create proper LP metadata
lpmake \
  --metadata-size 65536 \
  --super-name super \
  --metadata-slots 2 \
  --device super:${SUPER_SIZE} \
  --partition system:readonly:${SYSTEM_SIZE}:default \
  --image system=~/gsi/system_modified.img \
  --output ~/gsi/super_gsi_mod.bin

# OR if lpmake isn't available, the image can be assembled manually:
# The LP metadata at offset 0 maps the system partition starting at 0x100000

# 3. Pad to exact super partition size
truncate -s ${SUPER_SIZE} ~/gsi/super_gsi_mod.bin
```

### 4.6 Verify the Image

```bash
ls -l ~/gsi/super_gsi_mod.bin
# Should be exactly 7,134,339,072 bytes

# Verify LP metadata
python -c "
with open('super_gsi_mod.bin', 'rb') as f:
    f.seek(0x1000)
    magic = f.read(4)
    print(f'LP magic: {magic.hex()} (should be 67446c61)')
"
```

---

## Phase 5: Flash the Device

### 5.1 Enter BROM Mode

Same as Phase 2, Step 2.1.

### 5.2 Flash Super Partition

```bash
cd ~/mtkclient
source venv/bin/activate

# Flash the clean super image
python mtk.py w super ~/gsi/super_gsi_mod.bin

# This takes ~18-20 minutes at ~6 MB/s
# DO NOT INTERRUPT — an incomplete flash will require reflashing
```

### 5.3 First Boot

1. Unplug USB cable
2. Power on device normally (hold power button)
3. First boot takes 3-5 minutes — the system is optimizing apps
4. You should see the standard Android 12 setup wizard

---

## Phase 6: Post-Flash Configuration

### 6.1 Enable USB Debugging

1. Settings → About Phone → tap "Build Number" 7 times
2. Settings → Developer Options → Enable USB Debugging
3. Connect USB, authorize the PC on the device prompt

### 6.2 Disable Untrusted Vendor Apps

The vendor partition still contains apps from the malicious stock ROM:

```bash
ADB=~/Android/Sdk/platform-tools/adb

# Disable WhatsApp (from malicious vendor partition)
$ADB shell pm disable-user --user 0 com.whatsapp

# Disable X/Twitter (from malicious vendor partition) 
$ADB shell pm disable-user --user 0 com.twitter.android

# Install clean versions from Play Store if needed
```

### 6.3 Display & Performance Optimizations

```bash
# Set display resolution (optional, improves performance on low RAM)
$ADB shell wm size 720x1520
$ADB shell wm density 320

# Disable animations (snappier feel)
$ADB shell settings put global window_animation_scale 0.0
$ADB shell settings put global transition_animation_scale 0.0
$ADB shell settings put global animator_duration_scale 0.0

# Limit background processes
$ADB shell settings put global app_standby_enabled 1
```

### 6.4 Verify GPU Fix

```bash
$ADB shell getprop debug.renderengine.backend
# Should output: gles
```

---

## Phase 7: Security Verification

Run these checks to confirm the device is clean:

```bash
# SELinux enforcing
$ADB shell getenforce                    # → Enforcing

# Not rooted
$ADB shell which su                      # → (no output)

# Secure build
$ADB shell getprop ro.secure             # → 1
$ADB shell getprop ro.debuggable         # → 0

# Encrypted storage
$ADB shell getprop ro.crypto.state       # → encrypted

# No open network ports
$ADB shell ss -tlnp                      # → (no listeners)

# No accessibility hijacking
$ADB shell settings get secure enabled_accessibility_services  # → null

# No device admin/MDM
$ADB shell dumpsys device_policy | head -3  # → no active admins
```

---

## Troubleshooting

### Device won't enter BROM mode
- Ensure device is completely powered off (hold power 15 sec)
- Try different USB cable (short, data-capable)
- Try different USB port (USB 2.0 preferred over USB 3.0)
- Press Vol Up + Vol Down BEFORE plugging USB

### Flash fails with USB errors
- Use a short USB cable (< 1 meter)
- Remove USB hubs — connect directly to PC
- Try USB 2.0 port instead of USB 3.0

### Screen goes black / GPU crash after boot
- The GPU fix init script is missing or not working
- Reflash using `super_clean_gsi.bin` which has the fix baked in
- Verify: `adb shell getprop debug.renderengine.backend` should show `gles`

### Boot loops
- Ensure vbmeta was flashed with verification disabled
- Reflash both vbmeta and super partition

### "System has been destroyed" message
- Normal after flashing a GSI — proceed through the warning
- The device will boot normally after displaying this message

---

## File Checksums

```
# Verify image integrity before flashing:
sha256sum images/super_clean_gsi.bin
sha256sum images/vbmeta_disabled.img
```

---

## What Was Removed

The original Chinese stock ROM contained:
- Pre-installed apps with excessive permissions
- Unknown system services communicating with Chinese servers
- Modified versions of WhatsApp and X/Twitter (in vendor partition)
- Various Chinese-language bloatware

All of this is replaced by Google's official GSI, which contains only:
- Stock AOSP + Google Mobile Services
- Standard Google apps (Play Store, Chrome, Gmail, etc.)
- No third-party bloatware
