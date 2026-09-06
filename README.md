 Lenovo Tab P11 5G (TB-J607Z) — Android 16 GSI Guide

This guide documents the full process of flashing a Treble GSI running **Android 16** (crDroid 12.11, Doze-off's `crdroid_gsi_treble` build) onto a **Lenovo Tab P11 5G (TB-J607Z)** — Snapdragon 750G/480, dynamic partitions, VNDK 30 (Android 11 stock), arm64_binder64-ab, Project Treble compliant.

> ⚠️ **This is not an officially supported device for this GSI.** The vendor image ships with VNDK 30 (Android 11), several partitions are single-slot only, and radio/telephony HAL behavior is not guaranteed. Follow this guide at your own risk. A full, verified firmware backup is mandatory before you start.

## Table of Contents

- [Device Specs](#device-specs)
- [Prerequisites](#prerequisites)
- [Step 1 — Unlock Bootloader](#step-1--unlock-bootloader)
- [Step 2 — Full Partition Backup](#step-2--full-partition-backup)
- [Step 3 — Verify the Backup](#step-3--verify-the-backup)
- [Step 4 — Flash the GSI](#step-4--flash-the-gsi)
- [Step 5 — Wipe Userdata (⚠️ known pitfall)](#step-5--wipe-userdata--known-pitfall)
- [Step 6 — First Boot](#step-6--first-boot)
- [Step 7 — Fix Alarms & Notifications](#step-7--fix-alarms--notifications)
- [Step 8 — Root with APatch](#step-8--root-with-apatch)
- [Known Issues & Fixes](#known-issues--fixes)
- [Play Integrity / Root Hiding](#play-integrity--root-hiding)
- [Restore Procedure](#restore-procedure)
- [Credits](#credits)

## Device Specs

| Item | Value |
|---|---|
| Model | Lenovo Tab P11 5G (TB-J607Z) |
| SoC | Qualcomm Snapdragon 750G / 480 5G |
| Stock Android | 11 (ZUI), later OTA up to Android 13 |
| VNDK | 30.0, **not** VNDK-Lite |
| Partition layout | A/B, dynamic partitions (`super`), System-as-Root |
| Architecture | arm64, 64-bit binder |
| GSI used | [crDroid 12.11 GSI Treble (Android 16 QPR2)](https://github.com/Doze-off/crdroid_gsi_treble/releases) — `arm64-ab-GAPPS` variant |

Check your own device's Treble compliance with the **Treble Info** app (Play Store) before proceeding — VNDK version and partition scheme must match what's described above for these instructions to apply directly.

## Prerequisites

- Bootloader already unlocked (Lenovo unlock procedure, voids warranty).
- `adb` and `fastboot` (Android platform-tools) installed on your PC.
- USB cable, preferably a short, high-quality one — long/cheap cables have caused transfer hangs during large `fastboot flash` operations in testing.
- At least 15 GB of free space on your PC for backups.
- The GSI `.img.xz` file downloaded and extracted (`unxz filename.img.xz`).

> **Note on host tools:** distro-packaged `platform-tools` (e.g. Debian/Ubuntu `android-sdk-platform-tools`) can behave differently from Google's official platform-tools, notably around `fastboot -w` (see [Known Issues](#known-issues--fixes)). Google's official tools from `https://developer.android.com/tools/releases/platform-tools` are recommended.

## Step 1 — Unlock Bootloader

Standard Lenovo/Qualcomm unlock via `fastboot flashing unlock` or the Lenovo-provided unlock procedure. Not covered in detail here — this guide assumes it's already done.

## Step 2 — Full Partition Backup

**Do not skip this.** With root access (APatch/Magisk/KernelSU) already available on stock, list every partition:

```bash
adb shell su -c "ls -l /dev/block/by-name/"
```

Back up every partition using the on-device dump + `adb pull` method (this avoids PTY-related binary corruption that occurs with `adb exec-out` pipes through `su -c`):

```bash
#!/bin/bash
DEST=/path/to/backup
TMP=/data/local/tmp/backup
mkdir -p "$DEST"
adb shell su -c "mkdir -p $TMP"

PARTS="abl_a abl_b aop_a aop_b apdp bluetooth_a bluetooth_b boot_a boot_b \
cdt core_nhlos_a core_nhlos_b ddr devcfg_a devcfg_b devinfo dip \
dsp_a dsp_b dtbo_a dtbo_b featenabler_a featenabler_b frp fsc fsg \
hyp_a hyp_b imagefv_a imagefv_b keymaster_a keymaster_b keystore \
lenovocust lenovoraw limits mdtp_a mdtp_b mdtpsecapp_a mdtpsecapp_b \
metadata misc mlsp modem_a modem_b modemst1 modemst2 multiimgoem_a \
multiimgoem_b oemowninfo persist qupfw_a qupfw_b recovery_a recovery_b \
splash spunvm ssd storsec toolsfv tz_a tz_b uefisecapp_a \
uefisecapp_b uefivarstore vbmeta_a vbmeta_b vbmeta_system_a \
vbmeta_system_b xbl_a xbl_b xbl_config_a xbl_config_b"

for p in $PARTS; do
  echo "Backing up $p ..."
  adb shell su -c "dd if=/dev/block/by-name/$p of=$TMP/${p}.img bs=4M"
  adb pull "$TMP/${p}.img" "$DEST/${p}.img"
  adb shell su -c "rm $TMP/${p}.img"
done
adb shell su -c "rmdir $TMP"
```

### `super` partition (chunked backup)

`super` (10 GB on this device) is too large to stage entirely under `/data/local/tmp` on most units — chunk it:

```bash
CHUNK_BLOCKS=320   # 320 * 4M = 1.34 GB per chunk
CHUNKS=8           # adjust so CHUNKS * CHUNK_BLOCKS * 4M == total super size

for i in $(seq 0 $((CHUNKS-1))); do
  SKIP=$((i * CHUNK_BLOCKS))
  adb shell su -c "dd if=/dev/block/by-name/super of=$TMP/super_part${i}.img bs=4M skip=$SKIP count=$CHUNK_BLOCKS"
  adb pull "$TMP/super_part${i}.img" "$DEST/super_part${i}.img"
  adb shell su -c "rm $TMP/super_part${i}.img"
done
```

Reassemble on the PC:
```bash
cat super_part0.img super_part1.img ... super_part7.img > super.img
```

Get the exact `super` size first with `fastboot getvar partition-size:super` (in fastbootd) so your chunk math divides evenly.

## Step 3 — Verify the Backup

Compare every local file size against the real partition size before trusting the backup:

```bash
for f in *.img; do
  name="${f%.img}"
  local_size=$(stat -c %s "$f")
  remote_size=$(adb shell su -c "blockdev --getsize64 /dev/block/by-name/$name" 2>/dev/null | tr -d '\r')
  if [ "$local_size" == "$remote_size" ]; then
    echo "OK: $name ($local_size)"
  else
    echo "MISMATCH: $name local=$local_size remote=$remote_size"
  fi
done
sha256sum *.img > checksums.txt
```

All partitions must report `OK`. Do not proceed to flashing until this passes 100%.

## Step 4 — Flash the GSI

Prepare `vbmeta` from your own backup (reuse your device's signed images, just re-flash with the disable flags — no need for a custom vbmeta):

```bash
fastboot --disable-verity --disable-verification flash vbmeta vbmeta_a.img
fastboot --disable-verity --disable-verification flash vbmeta_system vbmeta_system_a.img
fastboot reboot fastboot
```

In fastbootd, flash the GSI directly to the logical `system` partition (fastboot auto-resizes it):

```bash
fastboot flash system crDroid-12.11_GSI_treble_arm64-ab-GAPPS-*.img
```

If you hit `Not enough space to resize partition`, free space on the **inactive** slot only (never touch the slot you're about to boot into):

```bash
fastboot delete-logical-partition product_b
fastboot delete-logical-partition system_ext_b
```

## Step 5 — Wipe Userdata (⚠️ known pitfall)

**Do not use `fastboot -w` blindly.** Some distro-packaged `fastboot` binaries attempt to pre-generate a local `userdata.img` with `make_f2fs`, and if that local tool call fails, `userdata` is left in an inconsistent state that causes repeated bootloops after first setup.

Use explicit erase commands instead, and let Android format the partition itself on first boot:

```bash
fastboot erase userdata
fastboot erase metadata
fastboot reboot
```

If you ever get stuck in a bootloop *after* initial setup completes once successfully, the most reliable fix is **not** re-running `fastboot -w`, but performing a factory reset from **within** the booted OS (`Settings → System → Reset → Erase all data`), which uses Android's own formatting routine instead of the host-side fastboot tool.

## Step 6 — First Boot

The first boot after wiping userdata can take 10–15 minutes. Do not interrupt it, even if it looks stuck on the boot animation.

## Step 7 — Fix Alarms & Notifications

If alarms and/or notifications silently fail to play sound after the first boot, open the **Treble Settings App** (the "Treble" app, `com.libremobileos.treble.settings`) on the device:

1. Go to **General**.
2. Enable **"use alternative audio policy"**.
3. Reboot the device.

This works around the vendor audio HAL not routing alarm/notification streams correctly under this GSI.

## Step 8 — Root with APatch

`boot` is untouched by the GSI flash (only `system` is replaced), so if the device was already rooted with APatch before backup, the same `boot_a.img` from your backup already contains the kernel patch — you may not need to re-patch at all.

If the APatch Manager app reports "Unknown" or "Not patched" (common after a userdata wipe, since the manager app itself lived in `/data`):

1. Install the APatch manager APK: `adb install APatch_x.x.x.apk`
2. Push your backed-up (or current) `boot_a.img` to the device: `adb push boot_a.img /sdcard/Download/`
3. In-app: **Patch to Image** → select the file → patch.
4. Pull the patched image back: `adb pull /sdcard/Download/patched_boot_a.img`
5. Flash it:
```bash
fastboot flash boot_a patched_boot_a.img
fastboot reboot
```

## Known Issues & Fixes

| Issue | Cause | Fix |
|---|---|---|
| Modem totally `OUT_OF_SERVICE` | Preferred network type set to "NR only" with no local Standalone-5G coverage | Switch to `NR/LTE/GSM/WCDMA` (auto fallback) |
| `adb exec-out ... | dd` produces oversized/corrupted `.img` files | PTY-based `su -c` allocates a pseudo-terminal, translating `0x0A` → `0x0D 0x0A` in binary data | Use `dd ... of=file` + `adb pull` instead of piping through a shell |
| `base64` decode "invalid input" | Same PTY corruption applies to base64-encoded streams too | Avoid the pipe-through-`su` approach entirely; use file + `adb pull` |
| `fastboot flash super`/large files: `No space left on device` | Internal storage (`/data/local/tmp`) too small for full 10 GB `super` dump | Chunk the dump into ~1.25 GB pieces (see Step 2) |
| Repeated bootloop after `fastboot -w` | Host `make_f2fs` binary fails silently, leaving `userdata` inconsistent | Use `fastboot erase userdata` + `fastboot erase metadata` instead of `-w`; or reset from within the booted OS |
| No 5G option in network type list (post-flash) | Normal — appears once modem registers with `isEnDcAvailable=true` at a covered location | Verify with `adb shell dumpsys telephony.registry \| grep -i mServiceState` |
| `fastboot flash boot_a` hangs indefinitely with "partition size: 0" warning | Fastboot AVB footer size lookup failing, sometimes correlates with a stuck USB transfer | Ctrl+C, unplug/replug USB, retry; try a different cable/port if it recurs |

## Play Integrity / Root Hiding

An unlocked bootloader + unofficial GSI will almost certainly fail `MEETS_DEVICE_INTEGRITY` and `MEETS_STRONG_INTEGRITY`. `MEETS_BASIC_INTEGRITY` is achievable with:

- **ZygiskNext** (APatch has no built-in Zygisk)
- **Play Integrity Fix** module
- Enable **"Exclude Modifications"** per-app in the APatch manager for extra hiding

Banking apps, Google Wallet, and anti-cheat-protected games will likely still fail regardless of hiding effort, since they typically require device/strong integrity.

## Restore Procedure

To revert to stock if anything goes irrecoverably wrong:

```bash
fastboot --disable-verity --disable-verification flash vbmeta vbmeta_a.img
fastboot --disable-verity --disable-verification flash vbmeta_system vbmeta_system_a.img
fastboot flash boot_a boot_a.img
fastboot flash dtbo_a dtbo_a.img
fastboot reboot fastboot
fastboot flash system system_a.img   # if you kept a raw dump of stock system, or restore the full `super.img`
fastboot erase userdata
fastboot erase metadata
fastboot reboot
```

Critical partitions (`xbl`, `abl`, `tz`, `hyp`, `xbl_config`) cannot be flashed via standard fastboot ("Flashing is not allowed for Critical Partitions") — this is expected and by design; they are never touched by this procedure and don't need restoring.

## Credits

- [Doze-off/crdroid_gsi_treble](https://github.com/Doze-off/crdroid_gsi_treble) — the GSI build used in this guide
- [phhusson/treble_experimentations](https://github.com/phhusson/treble_experimentations) — Project Treble GSI documentation and the `qcrilam` RIL compatibility shim (`me.phh.qcrilam`) that made cellular connectivity work on this device
- [APatch](https://github.com/bmax121/APatch) — kernel-based root solution used throughout this guide

---

*This guide reflects a real, documented flashing session on a specific TB-J607Z unit. Your mileage may vary depending on your exact firmware revision, carrier, and region.*
