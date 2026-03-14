# How I Did It — Cleaning a Malware-Infested "Welcome" Phone with AI

## The Short Version

I plugged a malware-infested Chinese phone into my Linux PC, installed ADB and the Android SDK, opened VS Code with **GitHub Copilot** (using the **Claude Opus 4.6** model), and told the AI what I wanted to accomplish. The AI handled every technical operation — from researching the device, to building the clean ROM image, to flashing it, to performing a complete security audit — all through terminal commands and file operations, with me providing direction and making decisions along the way.

---

## The Device

The **Welcome S17 Max Mini** (internal model **A25**) is a budget Android smartphone manufactured by **iBRIT**, a Chinese OEM. It was purchased from a Chinese seller on an online marketplace. The phone shipped with what appeared to be a standard Android 12 ROM, but the system partition was loaded with Chinese adware, spyware, and data-exfiltrating malware baked directly into the firmware.

**Key specs:**
- SoC: MediaTek MT6765 (Helio P35/G35)
- GPU: PowerVR Rogue GE8320
- RAM: ~2.9 GB
- Storage: eMMC with a 6.7 GB super partition (system + vendor)

## The Setup

The entire process was done on a Linux desktop. Here's what I set up before starting:

1. **Android SDK / Platform Tools** — Downloaded and installed the Android SDK, which provides `adb` (Android Debug Bridge) for communicating with the phone over USB
2. **USB Debugging** — Enabled Developer Options on the phone and turned on USB Debugging, then authorized the PC when prompted
3. **VS Code + GitHub Copilot** — Opened VS Code with the GitHub Copilot extension, using the **Claude Opus 4.6** model in agent mode

That's it. From there, I described what I wanted to achieve in plain English, and the AI took over the technical execution.

## The Process

### Phase 1: Discovery & Backup

The AI's first move was to connect to the device via ADB and map out the hardware — identifying the SoC (MT6765), GPU (PowerVR GE8320), partition layout, and the super partition structure (LP metadata format with system + vendor sub-partitions).

Before touching anything, we pulled full backups of all critical partitions (boot, lk, lk2, vbmeta, seccfg, recovery, super) using **mtkclient**, an open-source MediaTek BROM-level flash tool. This gave us a safety net in case anything went wrong — which it did.

### Phase 2: Building the Clean ROM

The approach was straightforward:
1. Download Google's official **Generic System Image (GSI)** — the arm64 Android 12 build with Google Mobile Services (`gsi_gms_arm64`)
2. Extract the system image from the GSI
3. Repack it into the device's super partition format (LP metadata), preserving the existing vendor partition layout
4. Flash the new super image via mtkclient at the BROM level

The AI handled the entire build pipeline: downloading the GSI, extracting it, using `lpmake` to construct the LP-format super image with the correct metadata (group names, partition sizes, block device mapping), and preparing the final flashable binary.

### Phase 3: The GPU Crash

The first flash went smoothly — the device booted into clean Android 12. But within seconds of reaching the home screen, the display would freeze or crash. The AI diagnosed this as a known incompatibility between the **PowerVR GE8320 GPU** and Android 12 GSI's default **Skia GL** rendering backend.

The fix: inject an init script into the system image that sets `debug.renderengine.backend=gles` at boot, forcing the GLES rendering backend instead. This required:
1. Mounting the system image read-write
2. Creating an `.rc` init script in `/system/etc/init/`
3. Repacking the super image

The AI built the modified image (`super_gsi_mod.bin`) — this is the image included in this kit.

### Phase 4: The First Brick

During the optimization phase (attempting swap and VM tuning), the first device was bricked — a USB error (-71) made it completely unrecoverable, even at the BROM level. The device was dead.

I obtained an identical replacement device from the same seller.

### Phase 5: Successful Flash

The replacement device was flashed with the known-good `super_gsi_mod.bin` (clean GSI + GPU fix only, no experimental mods). The flash was performed via mtkclient in BROM mode:

```
Power off → Hold Vol Up + Vol Down → Plug USB → mtkclient writes super partition
```

The flash completed at 100%, the device rebooted, and Android 12 came up clean — no malware, no Chinese apps, functional GPU, working Google Play Services.

### Phase 6: Security Audit

After the flash, I asked the AI to perform a thorough security audit of the vendor partition (which was retained from the original device, since it contains essential hardware drivers).

The AI:
- Pulled and verified the signing certificates of two pre-installed vendor apps (WhatsApp and X/Twitter) using `apksigner` — both turned out to be **genuine, legitimately signed** copies, just bundled as OEM bloatware
- Scanned all 78 vendor init scripts for network commands, hardcoded IPs, base64 payloads, and malware indicators — **found zero**
- Audited the SELinux policy — standard AOSP platform certs only
- Checked running processes — only 3 standard MediaTek HAL services
- Verified zero TCP connections at idle

Conclusion: the vendor partition is clean standard MediaTek firmware. The malware was entirely in the system partition, which has been completely replaced.

### Phase 7: Packaging

Finally, the AI organized everything into this redistributable kit — images, scripts, documentation, checksums, and a bundled copy of mtkclient — so anyone with the same device can flash it themselves.

## The AI's Role

To be clear about what the AI (GitHub Copilot with Claude Opus 4.6) actually did:

- **Ran all terminal commands** — ADB operations, mtkclient flashing, image building, file manipulation, security scanning
- **Wrote all scripts** — `flash.sh`, `post-flash-setup.sh`, `disable-vendor-apps.sh`
- **Wrote all documentation** — README, INSTRUCTIONS, this document
- **Performed all analysis** — partition mapping, APK certificate verification, init script scanning, SELinux policy review, process/network auditing
- **Made technical decisions** — choosing the GSI variant, diagnosing the GPU crash, selecting the rendering backend fix, structuring the super partition metadata
- **Handled errors and recovery** — when things went wrong (GPU crashes, bricked device, interrupted flash), the AI adapted and found solutions

My role was:
- **Direction** — telling the AI what I wanted to achieve ("clean this phone", "check if the vendor partition is safe", "package this up")
- **Decisions** — choosing when to proceed, when to abort, when to try a different approach
- **Physical interaction** — plugging in the USB cable, pressing buttons on the phone to enter BROM mode, confirming ADB authorization

## Tools Used

| Tool | Purpose |
|------|---------|
| **VS Code** | IDE / workspace |
| **GitHub Copilot** (Claude Opus 4.6) | AI agent — ran commands, wrote code, performed analysis |
| **ADB** (Android Debug Bridge) | Communicate with the phone over USB |
| **Android SDK Platform Tools** | ADB, fastboot |
| **Android SDK Build Tools** | `apksigner` for APK certificate verification |
| **mtkclient** | Open-source MediaTek BROM-level flash tool ([bkerler/mtkclient](https://github.com/bkerler/mtkclient)) |
| **lpmake** | Build LP-format super partition images |
| **Google GSI** | Official Generic System Image (Android 12 arm64 with GMS) |

## Timeline

This entire process — from first connecting the malware-infested phone to having a fully clean, audited, documented ROM kit — took place over a couple of days in March 2026. One device was bricked and replaced along the way. The AI handled hundreds of individual terminal commands and file operations across the process.

## Lessons Learned

1. **Budget Chinese phones are genuinely dangerous** — the malware was baked into the system firmware, not just side-loaded apps. A factory reset would not remove it.
2. **PowerVR GPUs need special handling on GSI** — the Skia GL backend crash is a known issue that's easy to fix once you know about it, but will brick your display otherwise.
3. **Vendor partitions are usually fine** — the malware lives in the system partition. The vendor partition is typically just standard SoC firmware from MediaTek/Qualcomm/etc.
4. **BROM mode is your safety net** — on MediaTek devices, as long as you can enter BROM mode (Vol Up + Vol Down + USB), you can always reflash. The device is only truly bricked if BROM stops responding.
5. **AI can handle complex hardware operations** — with the right tools installed (ADB, mtkclient) and a human making the high-level decisions, an AI agent can execute the entire technical pipeline of ROM building, flashing, and verification.
