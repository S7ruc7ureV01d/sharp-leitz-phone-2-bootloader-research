# SHARP Leitz Phone 2 (LP-02) — Bootloader Unlock Research

Personal, non-destructive research into whether the bootloader of a SHARP "Leitz Phone 2" (SoftBank Japan model `LP-02`) can be unlocked, or root otherwise obtained, despite the carrier disabling OEM unlock. This is device-ownership research on my own hardware — not an attack on any third party, and no exploit tooling is published here (see [Responsible disclosure](#responsible-disclosure-notes) below).

**Status:** Bootloader locked, device unmodified. All investigation so far has been read-only against the device (property inspection, package inspection, boot-mode enumeration). Nothing has been flashed, wiped, or written.

---

## 1. Device identification

| Property | Value |
|---|---|
| Marketing name | SHARP Leitz Phone 2 |
| Model / product | `LP-02` |
| Device / board | `MinevaL` / `Mineva` |
| Manufacturer | SHARP (shares lineage with the SHARP AQUOS R7/R8 family) |
| Carrier | **SoftBank (Japan)** — confirmed via `jp.softbank.*` / `jp.co.softbank.*` system packages present on the stock ROM |
| SoC | Qualcomm SM8450 (Snapdragon 8 Gen 1) |
| Android version (current) | 13, build `S8019` / `02.00.20` |
| Framework security patch | 2023-11-01 |
| Fingerprint | `SG/LP-02/MinevaL:13/S8019/02.00.20:user/release-keys` |

This is the Japan-only, SoftBank-branded "Leitz Phone 2" (Leica co-engineered, AQUOS R8 pro derivative) — not a global retail unit. That matters because SoftBank Japan devices are historically carrier-locked against bootloader unlock as a matter of policy.

## 2. Confirmed lock state

```
ro.boot.flash.locked=1
ro.boot.vbmeta.device_state=locked
ro.boot.verifiedbootstate=green
ro.boot.avb_version=1.2
sys.oem_unlock_allowed=0
ro.boot.force_normal_boot=1
```

**Root cause of the greyed-out "OEM unlocking" toggle:** confirmed via `adb shell service call oem_lock`, which surfaces a `SecurityException` stack trace naming the real check:

```
com.android.server.oemlock.OemLockService.enforceManageCarrierOemUnlockPermission(...)
"Can manage OEM unlock allowed by carrier: Neither user 2000 nor current process has android.permission.MANAGE_CARRIER_OEM_UNLOCK_STATE"
```

This is standard AOSP `OemLockService` / `PersistentDataBlockService` behavior: `isOemUnlockAllowedByCarrier()` returns **false**, set by SoftBank/SHARP at manufacture and stored in the FRP persistent partition (`ro.frp.pst=/dev/block/bootdevice/by-name/frp`). The on-device "Connect to the internet or contact your carrier" message is generic AOSP text — the lock is **not** actually gated on connectivity; it's a static carrier flag baked in at manufacture.

No root/su is available; adb is unprivileged shell (uid 2000) and cannot be escalated (`adbd cannot run as root in production builds`). Raw partition reads (`frp`, `devinfo`, `misc`) as shell all return `Permission denied`.

## 3. Boot-mode behavior (tested)

| Command | Result |
|---|---|
| `adb reboot bootloader` | Boots straight back into Android. No fastboot USB interface ever appears (`fastboot devices` stays empty). |
| `adb reboot abl` | Same — boots straight back into Android, no interactive ABL/fastboot prompt reachable over USB. |
| `adb reboot edl` | **Works.** Phone drops to Qualcomm EDL immediately, enumerates as USB `05c6:9008` ("QDL mode"). |

`ro.boot.force_normal_boot=1` means SHARP's ABL (the fastboot-capable stage) is configured to always chain straight to normal Android boot rather than stopping at an interactive fastboot menu. **EDL is different** — it's entered from the SoC's Primary Bootloader (PBL), which lives in Boot ROM beneath ABL/TZ/anything SHARP or SoftBank configures in software. `force_normal_boot` has no effect on it, which is why EDL is reachable even though fastboot is not.

EDL access alone is **not** root or unlock — it only starts a Sahara handshake, which refuses to hand control to a Firehose programmer unless it's signed by Qualcomm/SHARP's keys (secure boot / PBL signature verification, consistent with `verifiedbootstate=green`). No signed SHARP firehose loader for this device is in hand, so EDL alone doesn't currently grant flash read/write.

## 4. Landscape of known unlock methods

- **No SHARP AQUOS device has a standard fastboot unlock path.** Cross-referenced against a public "bootloader-unlock wall of shame" — SHARP is listed as a brand that removes/refuses `fastboot oem unlock` entirely, even when "OEM unlocking" is toggled on in Developer Options.
- **Balmuda Phone (A101BM)**, another SoftBank-Japan device, has a documented XDA report of the identical symptom: OEM unlocking enabled in Settings, `fastboot flashing unlock` still fails. No resolution found there either.
- **Hikari Calyx Tech** ran a one-time, now-dormant free unlock campaign (May 2022) for a narrow list of older FIH-manufactured Nokia/Sharp devices. The Leitz Phone 2 / AQUOS R8 family is not on that list.
- **No dedicated public forum thread exists yet** for "Leitz Phone 2," "AQUOS R8 pro," or the "Mineva"/"MinevaL" codename.
- **No hidden engineering menu found for bootloader unlock.** One SHARP-specific dialer secret code exists on-device — `*#*#7390192#*#*` — which targets `jp.co.sharp.android.shselfcheck` (a self-test/service menu), but the app's UI appears gated behind an engineering/debug build check that's compiled out on this production (`user`/`release-keys`) build: the broadcast is consumed silently and no menu appears.

## 5. Lead A: CVE-2026-25262 (Qualcomm Sahara write-what-where)

A third-party researcher's public repository (referenced, not mirrored here) documents a vulnerability in the Sahara handshake stage — the protocol EDL/9008 mode speaks *before* a Firehose loader is authenticated — allowing arbitrary SRAM writes while the PBL is still verifying the loader's signature.

The researcher experimentally confirmed this **specifically on SM8450 (Snapdragon 8 Gen 1)** — the same SoC as this phone — using a different device as testbed. This extends the CVE beyond its originally-acknowledged affected chipset list.

**Caveats (as of the referenced research's last update):** only partial success was achieved — an injected Firehose responds to `nop` without hitting the normal auth-error path, but full UFS storage access was not obtained (`getstorageinfo` and similar return empty, meaning TrustZone/Secure World handoff likely never completes). No public working exploit exists for this yet; it's active, unfinished research and not currently a usable path to flash read/write, root, or bootloader unlock on this device.

## 6. Lead B: vendor kernel patch gap (root without unlock)

```
Kernel:         5.10.136-android12-9-00026-gfb4b7271c902-ab9710341 (built 2023-03-08)
Framework SPL:  2023-11-01
Vendor SPL:     2022-02-05   <-- ~21 months behind the framework SPL
ro.build.type:  user (release-keys, not debuggable)
```

SHARP patches the Android framework regularly but has left vendor/kernel-driver components (GPU, DSP, camera HAL) frozen since **February 2022**. That's a measurable attack surface: any Qualcomm vendor-side CVE disclosed after Feb 2022 is a candidate for still being present, unpatched, on this device. This targets **root via a local kernel exploit reachable from an unprivileged adb shell — no bootloader unlock required.**

- **CVE-2022-0847 ("Dirty Pipe")** — upstream fixed 2022-02-20 in kernel 5.10.102. This device's kernel (5.10.136) is version-numerically above that fix point, but vendor forks routinely diverge from mainline numbering and don't automatically inherit backports. The vendor SPL (2022-02-05) predates the public fix by two weeks — circumstantial evidence this device may have shipped without the fix. Not confirmed without direct testing.
- **CVE-2022-22071** — Qualcomm FastRPC (aDSP) driver local privilege escalation, patched by Qualcomm May 2022. Vendor SPL here predates this fix by 3 months.
- **CVE-2023-33106 / CVE-2023-33107** — Qualcomm Adreno GPU (KGSL) driver bugs (this phone uses Adreno 730), both used in real targeted spyware attacks per Google TAG/Project Zero, patched by Qualcomm mid-2023. Vendor SPL here is far behind. A public PoC exists for CVE-2023-33107 targeting a *different* SoC (Snapdragon 855 / SM7250) — the underlying KGSL driver bug is shared Qualcomm kernel code, so porting to SM8450/Adreno 730 is architecturally plausible but requires real offset/structure-layout analysis against this device's actual kernel image; it is not a drop-in exploit.

This vendor-patch-gap angle is currently the most concrete, actionable lead in this investigation.

## 7. Next steps

1. Retry the SHARP self-check code (`*#*#7390192#*#*`) by hand-dialing directly on the device rather than via ADB key injection, to rule out an input-delivery issue vs. a build-gated feature.
2. Track upstream research on the Sahara write-what-where CVE for SM8450 in case full UFS initialization is achieved.
3. Pull this device's kernel image and compare KGSL driver structure offsets against the public CVE-2023-33107 writeup to scope out a port.
4. Investigate whether a UART/EDL hardware test point exists on this board (undocumented publicly for Mineva/LP-02 so far).

## Responsible disclosure notes

This repository documents *analysis of already publicly-disclosed, already-patched-upstream* vulnerabilities (CVE-2022-0847, CVE-2022-22071, CVE-2023-33106/33107, CVE-2026-25262) as they may apply to a specific device's outdated vendor firmware. No new vulnerability is disclosed here, and no exploit code, PoC, or offensive tooling is included or linked in runnable form. The goal is personal device unlock/root, not attacking third-party systems.
