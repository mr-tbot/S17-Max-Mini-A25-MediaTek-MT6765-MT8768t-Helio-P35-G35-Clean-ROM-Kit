# Welcome / Iku S17 Max Mini / A25 — Clean ROM Kit
Join the official XDA forum on this topic: 
https://xdaforums.com/t/welcome-iku-s17-max-mini-a25-mt6765-my8768t-helio-p35-g35-clean-stock-android-12-rom-with-optional-root-kit.4782153/

### MediaTek MT6765/MT8768t (Helio P35/G35) | Android 12 GSI | GPU Fix Included

![S17-Max-Mini](S17-max-mini.png?raw=true "S17-Max-Mini")

> **One-line summary:** Replaces the malware-infested Chinese stock firmware with a clean Google Android 12 Generic System Image, including a critical GPU fix for the PowerVR GE8320.

---

## Table of Contents

- [Download](#download)
- [About This Device](#about-this-device)
- [Why This Exists](#why-this-exists)
- [What's In This Kit](#whats-in-this-kit)
- [Hardware Specifications](#hardware-specifications)
- [Prerequisites](#prerequisites)
- [Back Up Your Device First](#back-up-your-device-first)
- [Flashing Guide (Step-by-Step)](#flashing-guide-step-by-step)
  - [Step 1 — Install Dependencies](#step-1--install-dependencies)
  - [Step 2 — Verify Image Integrity](#step-2--verify-image-integrity)
  - [Step 3 — Enter BROM Mode](#step-3--enter-brom-mode)
  - [Step 4 — Flash the Clean ROM](#step-4--flash-the-clean-rom)
  - [Step 5 — First Boot](#step-5--first-boot)
  - [Step 6 — Post-Flash Setup](#step-6--post-flash-setup)
- [Verifying Your Flash](#verifying-your-flash)
- [GPU Fix Explained](#gpu-fix-explained)
- [Vendor Partition & Security Audit](#vendor-partition--security-audit)
- [Backup & Restore](#backup--restore)
- [Rooting (Optional)](#rooting-optional)
- [Troubleshooting](#troubleshooting)
- [Dialer Codes (Stock ROM Only)](#dialer-codes-stock-rom-only)
- [How This ROM Was Built](#how-this-rom-was-built)
- [Support / Donate](#support--donate)
- [Disclaimer](#disclaimer)
- [License](#license)

---

## Download

> **The full kit (~7 GB) is too large for GitHub.** Download it from Google Drive:

### [**Download Clean ROM Kit (Google Drive)**](https://drive.google.com/drive/folders/1QDIGHy6e0DY7Sg-XCp5KcUlZ7gw4OT85?usp=sharing)

The link contains a ZIP archive with everything — flash images, scripts, backups, tools, and documentation.

**After downloading:**

```bash
# 1. Extract the archive
unzip "Welcome_S17_Max_Mini_Clean_ROM_Kit.zip"

# 2. Verify integrity (compare against CHECKSUMS.sha256)
cd "Welcome S17 Max Mini"*
sha256sum -c CHECKSUMS.sha256
```

All checksums should report `OK`. **Do not flash if any checksum fails.**

---

## About This Device

The **Welcome S17 Max Mini** (internal model **A25**) is a budget Android smartphone manufactured by **iBRIT**, a Chinese OEM that produces devices under various brand names. These devices are commonly found on AliExpress, Banggood, and similar marketplaces.

| Field | Value |
|---|---|
| Marketing Name | Welcome S17 Max Mini |
| Internal Model | A25 |
| Manufacturer | iBRIT |
| Brand (as shipped) | Welcome / iku |
| Build Fingerprint | `iku/A25/A25:12/SP1A.210812.016/mp1V15240:user/release-keys` |
| Vendor Build Date | December 18, 2025 |
| Country of Origin | China |

The device ships with what appears to be a legitimate Android 12 ROM, but **the system partition is loaded with Chinese adware, spyware, and data-exfiltrating malware** baked directly into the firmware. A factory reset does **not** remove it. This kit replaces the compromised system partition entirely.

---

## Why This Exists

Many budget MT6765 devices — including this Welcome S17 Max Mini — are sold on AliExpress and similar marketplaces with Chinese ROMs containing **pre-installed malware, spyware, and data-exfiltrating apps baked into the system firmware**. These cannot be removed through normal means (factory reset, app uninstall) because they are part of the system image itself.

This kit replaces the entire system partition with **Google's official Generic System Image (GSI)**, providing a clean, malware-free Android 12 experience while retaining the original hardware drivers so everything still works.

---

## What's In This Kit

```
├── README.md                     ← You are here
├── INSTRUCTIONS.md               ← Detailed technical build log
├── HOW-I-DID-IT.md               ← The full story of how this ROM was created
├── ROOTING-GUIDE.md              ← Optional rooting via Magisk
├── CHECKSUMS.sha256              ← SHA256 checksums for verification
│
├── images/
│   ├── super_clean_gsi.bin       ← READY-TO-FLASH clean super image (6.7 GB)
│   ├── vbmeta_disabled.img       ← Disabled vbmeta (required for GSI boot)
│   └── gsi_gms_arm64-12.0-*.zip  ← Original Google GSI download (reference)
│
├── scripts/
│   ├── flash.sh                  ← Automated flash script (does everything)
│   ├── post-flash-setup.sh       ← Post-flash optimization & security checks
│   ├── disable-vendor-apps.sh    ← Disable untrusted vendor bloatware
│   └── a25_backup_script.sh      ← Full partition backup script
│
├── backup/
│   └── stock_partitions/         ← Original stock partition backups
│       ├── boot.bin              ← Stock boot image
│       ├── lk.bin / lk2.bin      ← Stock bootloaders
│       ├── vbmeta.bin            ← Stock vbmeta
│       ├── recovery.bin          ← Stock recovery
│       ├── seccfg.bin            ← Security config
│       └── gpt.bin               ← Partition table
│
└── tools/
    └── mtkclient/                ← MediaTek BROM flash tool (bundled)
```

---

## Hardware Specifications

| Field | Value |
|---|---|
| SoC | MediaTek MT6765 / MT8768t (Helio P35/G35) |
| GPU | PowerVR Rogue GE8320 |
| RAM | ~2.9 GB |
| Storage | eMMC |
| Super Partition | 7,134,339,072 bytes (6.64 GB) |
| Clean ROM | Android 12 GSI (SQ3A.220705.003.A1) with GMS |
| Architecture | arm64 |

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Operating System** | Linux (Ubuntu/Debian recommended). These instructions are Linux-only. |
| **Python** | Python 3.8 or newer |
| **USB Cable** | Data-capable USB cable (short cable preferred, < 1 meter) |
| **ADB** | Android SDK Platform Tools ([download](https://developer.android.com/tools/releases/platform-tools)) or `sudo apt install android-tools-adb` |
| **Disk Space** | ~15 GB free (for images + backups) |

---

## Back Up Your Device First

> **Always back up your device before flashing — even though this kit includes stock partition backups, backing up your own device is essential best practice.**

Flashing is a destructive operation. If something goes wrong, having your own backup means you can always restore your specific device to its original state.

### What to Back Up

1. **Personal data** — photos, contacts, messages, app data. Copy anything important off the device before starting.

2. **Full partition backup via mtkclient** — this captures every partition on the device at the BROM level:

```bash
# Put device in BROM mode (power off → Vol Up+Down → plug USB)
cd tools/mtkclient
pip install -r requirements.txt

# Back up ALL partitions (takes ~30 minutes)
python3 mtk.py rl ~/my_a25_backup --skip userdata
```

Or use the included backup script for a guided backup:

```bash
chmod +x scripts/a25_backup_script.sh
./scripts/a25_backup_script.sh
```

3. **Verify your backups exist** before proceeding:

```bash
ls -lh ~/my_a25_backup/
# You should see: boot.bin, super.bin, vbmeta.bin, lk.bin, lk2.bin, recovery.bin, etc.
```

### Why Back Up Even Though Stock Images Are Included?

- The stock backups in this kit are from **one specific device** — yours may have slightly different firmware versions or calibration data
- Partitions like `nvram` and `nvcfg` contain **device-unique data** (IMEI, Wi-Fi/Bluetooth MAC, RF calibration) that cannot be restored from another device's backup
- If BROM mode stops working (rare but possible), having a pre-flash backup is your only recovery option

> **Bottom line:** Spend the 30 minutes on a full backup. You will not regret it.

---

## Flashing Guide (Step-by-Step)

> **WARNING:** Flashing will erase all data on the device. Back up anything you need first (see [Back Up Your Device First](#back-up-your-device-first)).

### Step 1 — Install Dependencies

**Install mtkclient** (the MediaTek BROM flash tool, bundled in this kit):

```bash
cd tools/mtkclient
pip install -r requirements.txt
```

**Set up USB permissions** (Linux — required for mtkclient to access the device):

```bash
echo 'SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d", MODE="0666"' | sudo tee /etc/udev/rules.d/51-mediatek.rules
sudo udevadm control --reload-rules
sudo udevadm trigger
```

**Install ADB** (if not already installed):

```bash
sudo apt install android-tools-adb
# Or download from https://developer.android.com/tools/releases/platform-tools
```

### Step 2 — Verify Image Integrity

Before flashing, verify the images haven't been corrupted or tampered with:

```bash
cd /path/to/this/kit
sha256sum -c CHECKSUMS.sha256
```

All checksums should report `OK`. **Do not flash if any checksum fails.**

### Step 3 — Enter BROM Mode

BROM (Boot ROM) mode allows low-level access to the device's flash storage via USB:

```
┌─────────────────────────────────────────────────────────┐
│  1. Power OFF the device completely (hold power 15 sec) │
│  2. Hold VOLUME UP + VOLUME DOWN simultaneously         │
│  3. While holding both buttons, plug in the USB cable   │
│  4. Keep holding until the flash tool detects the device│
└─────────────────────────────────────────────────────────┘
```

Verify the device is detected:

```bash
lsusb | grep -i mediatek
# Should show: "MediaTek Inc." with ID 0e8d:0003
```

### Step 4 — Flash the Clean ROM

**Option A: Automated (recommended)**

The flash script handles everything — backup, bootloader unlock, vbmeta flash, and super partition flash:

```bash
chmod +x scripts/flash.sh
./scripts/flash.sh
```

The script will:
1. Verify all images and prerequisites
2. Wait for the device in BROM mode
3. Back up critical partitions (boot, vbmeta, lk, lk2, seccfg)
4. Unlock the bootloader
5. Flash `vbmeta_disabled.img` (disables verified boot for GSI compatibility)
6. Flash `super_clean_gsi.bin` (the clean Android 12 system image, ~18-20 min)

> **CRITICAL:** Do **NOT** disconnect the USB cable or close the terminal during the flash. An interrupted flash can brick the device.

**Option B: Manual**

If you prefer to run each step yourself:

```bash
cd tools/mtkclient

# 1. Unlock the bootloader
python3 mtk.py da seccfg unlock

# 2. Flash disabled vbmeta (allows unsigned system image to boot)
python3 mtk.py w vbmeta ../../images/vbmeta_disabled.img

# 3. Flash the clean super image (~18-20 minutes)
python3 mtk.py w super ../../images/super_clean_gsi.bin
```

### Step 5 — First Boot

1. **Unplug** the USB cable
2. **Power on** the device (hold power button) (it will now complain that it is bootloader unlocked and you need to press any button on the device when the warning pops up to get it to boot - this is normal behavior on these devices when their bootloader is unlocked. - if you don't the phone will shut back off and not boot)
3. **Wait 3-5 minutes** — the first boot is slow as Android optimizes apps
4. You may see a **"System has been destroyed"** warning — this is normal for GSI installs, just proceed
5. Complete the **Android setup wizard**

> **If the device does not boot or gets stuck on the logo:** You may need to perform a factory reset before the device will boot the new ROM. Enter recovery mode (power off → hold **Volume Up + Power** until the recovery menu appears), select **"Wipe data / factory reset"**, confirm, then reboot. This is common when flashing a new system image over an existing installation.

### Step 6 — Post-Flash Setup

After completing the Android setup wizard:

1. **Enable USB Debugging:**
   - Settings → About Phone → tap **Build Number** 7 times
   - Settings → Developer Options → enable **USB Debugging**
   - Connect USB and authorize the PC when prompted on the device

2. **Run the post-flash setup script:**

```bash
chmod +x scripts/post-flash-setup.sh scripts/disable-vendor-apps.sh
./scripts/post-flash-setup.sh
```

This script will:
- Verify the GPU fix is active
- Set optimal display resolution (720x1520 @ 320dpi)
- Disable animations for snappier performance
- Disable untrusted vendor apps (WhatsApp/X from the malicious ROM)
- Run a security verification checklist

3. **Disable vendor bloatware** (if not already done by the above script):

```bash
./scripts/disable-vendor-apps.sh
```

---

## Verifying Your Flash

After completing the flash and setup, verify everything is correct:

```bash
# GPU fix active
adb shell getprop debug.renderengine.backend
# Expected: gles

# SELinux enforcing
adb shell getenforce
# Expected: Enforcing

# Not rooted
adb shell which su
# Expected: (no output)

# Encrypted storage
adb shell getprop ro.crypto.state
# Expected: encrypted

# No open network ports
adb shell ss -tlnp
# Expected: (no listeners)

# Secure build
adb shell getprop ro.secure
# Expected: 1
```

### Post-Flash Security State

| Check | Expected Value |
|---|---|
| SELinux | Enforcing |
| Root access | None (no `su` binary) |
| Storage encryption | Encrypted |
| ADB | Requires authorization |
| Network listeners at idle | None |
| Debuggable | No (`ro.debuggable=0`) |

---

## GPU Fix Explained

The **PowerVR GE8320 GPU** in MT6765 devices crashes when Android 12 GSI's SurfaceFlinger uses the default **Skia GL** rendering backend. Without this fix, the display freezes or crashes within seconds of booting.

**The fix:** An init script baked into the system image forces the GLES rendering backend:

```
# /system/etc/init/gpu-fix.rc
on early-init
    setprop debug.renderengine.backend gles

on boot
    setprop debug.renderengine.backend gles
```

This is already included in `super_clean_gsi.bin` — no manual action required.

---

## Vendor Partition & Security Audit

This clean ROM replaces only the **system** partition. The **vendor** partition (containing hardware drivers, HAL services, and firmware for the MT6765 SoC) is retained from the original device and is required for the hardware to function.

### Pre-installed Vendor Apps

The vendor partition includes two pre-installed apps bundled by the phone manufacturer:

| App | Version | Size | Verdict |
|---|---|---|---|
| WhatsApp | v2.25.36.74 | 135 MB | Legitimate (signed by `CN=Brian Acton, O=WhatsApp Inc.`) |
| X / Twitter | v11.7.0 | 222 MB | Legitimate (signed by `CN=Leland Rechis, O=Twitter, Inc.`) |

A full security audit was performed on the vendor partition:

- **APK signatures** — verified via `apksigner`; both apps are signed by their official developers with valid v1/v2 certificate chains
- **APK contents** — standard DEX files and native libraries only; no suspicious binaries or hidden executables
- **Vendor init scripts** — all 78 `.rc` files scanned; zero network commands, zero hardcoded IPs, zero base64 payloads, zero malware indicators
- **SELinux policy** — standard AOSP platform certificates; vendor `seapp_contexts` contains only 3 standard MediaTek entries
- **Runtime audit** — only 3 vendor HAL services running (mtkpower, nvram, pq); zero TCP connections at idle
- **Google overlays** — 6 configuration overlay APKs, all signed by the standard AOSP platform key

**Despite being legitimate, these apps should still be disabled** — they came from the malicious ROM's vendor image and cannot be fully trusted. Install fresh copies from Google Play if needed:

```bash
./scripts/disable-vendor-apps.sh
```

---

## Backup & Restore

### Backing Up Before Flashing

The `flash.sh` script automatically backs up critical partitions before flashing. Backups are saved to `backup/pre_flash_<timestamp>/`.

To manually do a full backup of all partitions:

```bash
chmod +x scripts/a25_backup_script.sh
./scripts/a25_backup_script.sh
```

### Restoring to Stock

If you need to restore the original stock firmware, use the backups in `backup/stock_partitions/`:

```bash
cd tools/mtkclient

# Put device in BROM mode first, then:
python3 mtk.py w boot     ../../backup/stock_partitions/boot.bin
python3 mtk.py w lk       ../../backup/stock_partitions/lk.bin
python3 mtk.py w lk2      ../../backup/stock_partitions/lk2.bin
python3 mtk.py w vbmeta   ../../backup/stock_partitions/vbmeta.bin
python3 mtk.py w recovery ../../backup/stock_partitions/recovery.bin
python3 mtk.py w seccfg   ../../backup/stock_partitions/seccfg.bin
```

> **Note:** The stock super partition backup is not included in this kit due to size (~6.7 GB). The backup script can capture it if run before flashing.

---

## Rooting (Optional)

If you want root access via **Magisk**, see the full guide:

**[ROOTING-GUIDE.md](ROOTING-GUIDE.md)**

Summary of the process:
1. Extract the boot image from the device
2. Patch it with the Magisk app
3. Flash the patched boot image via mtkclient

> **Note:** Rooting reduces security. The main flash kit does **not** root the device.

---

## Troubleshooting

### Device won't enter BROM mode
- Ensure the device is **completely** powered off (hold power 15 seconds)
- Try a different USB cable (must be data-capable, not charge-only)
- Try a different USB port (USB 2.0 often works better than USB 3.0)
- Press Volume Up + Volume Down **before** plugging in USB

### Flash fails with USB errors
- Use a **short USB cable** (< 1 meter)
- Remove USB hubs — connect directly to the PC
- Try a USB 2.0 port instead of USB 3.0
- On Linux, ensure the udev rules are in place (see [Step 1](#step-1--install-dependencies))

### Screen goes black / GPU crash after boot
- The GPU fix may not be active
- Reflash using `super_clean_gsi.bin` (it has the fix baked in)
- Verify: `adb shell getprop debug.renderengine.backend` should output `gles`

### Boot loops
- Ensure vbmeta was flashed with verification disabled
- Reflash both `vbmeta_disabled.img` and `super_clean_gsi.bin`

### Device won't boot / stuck on logo after flash
- The device may require a **factory reset** before it will boot the new ROM
- Enter recovery mode: power off → hold **Volume Up + Power** until the recovery menu appears
- Select **"Wipe data / factory reset"** → confirm → reboot
- This is common when flashing a new system image over an existing data partition

### "System has been destroyed" message
- **This is normal** after flashing a GSI — not an error
- The device will boot normally after this message

### "No command" screen in recovery
- Power off and boot normally (just hold power button)
- If stuck, re-enter BROM mode and reflash

### mtkclient not detecting device
- Check `lsusb | grep -i mediatek` — you should see `0e8d`
- If not, the device isn't in BROM mode; try again
- Run mtkclient with `sudo` if USB permissions aren't configured

---

## Dialer Codes (Stock ROM Only)

These hidden dialer codes work on the **original stock Chinese ROM** (before flashing). They do **not** work after flashing the clean GSI.

| Code | Function |
|---|---|
| `*#081355#` | Configuration menu |
| `*#132978*#` | Configuration menu |
| `*#132686*#` | Configuration menu |

> These codes are rotated by the manufacturer and may vary between devices/firmware versions.

---

## How This ROM Was Built

The entire process — from identifying the malware to building and flashing the clean ROM — was performed using **GitHub Copilot** (Claude Opus 4.6) as an AI coding agent, running terminal commands, writing scripts, and performing security analysis, with a human providing direction and physical device interaction.

Basically - install VSCODE - pay for Copilot Pro - plug in the device to the machine - inform copilot that the phone is plugged in and provide it my instructions.md file - ask it to install necessary things like the Android SDK, ADB / Platform toolks, and any relevant MediaTek tools - and ask it to backup the original images first and then interactively help you clean and make a GSI rom for the device.  It took some trial and error and one strange bricked device - so proceed with caution - but the latest Claude and VSCODE made this incredibly easy to do!  You can also use this method on any Android device to diagnose and clean up any device - just enable ADB debugging over USB and let'er rip!

For the full technical story, see:
- **[HOW-I-DID-IT.md](HOW-I-DID-IT.md)** — narrative of the entire process
- **[INSTRUCTIONS.md](INSTRUCTIONS.md)** — detailed technical build log with every command

### Build Process Summary

1. **Backup** all stock partitions via mtkclient (BROM mode)
2. **Download** Google's official GSI (`gsi_gms_arm64` Android 12)
3. **Inject** GPU fix init script into the system image (`debug.renderengine.backend=gles`)
4. **Repackage** into the device's super partition format using `lpmake` (LP metadata)
5. **Flash** via mtkclient: unlock bootloader → disable vbmeta → write super partition
6. **Audit** the retained vendor partition for security

---

## Scripts Reference

| Script | Purpose | When to Run |
|---|---|---|
| `scripts/flash.sh` | Full automated flash (backup → unlock → flash vbmeta → flash super) | During flash |
| `scripts/post-flash-setup.sh` | GPU verification, display config, disable animations, security checks | After first boot + setup wizard |
| `scripts/disable-vendor-apps.sh` | Disable WhatsApp & X/Twitter from the malicious vendor ROM | After first boot |
| `scripts/a25_backup_script.sh` | Full partition backup of all device partitions | Before flash (optional) |

---

## Support / Donate

If this kit saved your device (or your sanity), consider supporting the development of [MR-TBOT.com](https://mr-tbot.com) and [Codedatda.casa](https://codedatda.casa) projects. Your donations help fund continued work on open-source tools, device research, and guides like this one.

### [**Donate via PayPal**](https://www.paypal.com/donate/?business=7DQWLBARMM3FE&no_recurring=0&item_name=Support+the+development+and+growth+of+innovative+MR_TBOT+projects.&currency_code=USD)

Every contribution — no matter the size — is appreciated and goes directly toward keeping these projects alive.

---

## Disclaimer

**USE AT YOUR OWN RISK.** This ROM kit is provided as-is, with no warranties of any kind, express or implied. The author(s) take **no responsibility** for any damage to your device, data loss, bricked hardware, voided warranties, or any other issues arising from the use of this ROM, these tools, or any information provided in this kit.

Flashing custom firmware carries inherent risk — you are solely responsible for your device and your decision to proceed.

---

## License

This kit documents a process using publicly available tools. No proprietary code is distributed.

- **Google GSI** — [Android Open Source Project](https://developer.android.com/topic/generic-system-image) (Apache 2.0)
- **mtkclient** — [bkerler/mtkclient](https://github.com/bkerler/mtkclient) (GPLv3)
- **Scripts & Documentation** — provided freely for anyone with the same device
