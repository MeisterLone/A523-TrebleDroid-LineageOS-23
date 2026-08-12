# From Forced OTA to Android 16: Reclaiming an Allwinner A523 Tablet

This repository documents a hobby research and learning exercise: taking an inexpensive, obscure, and frustratingly
locked-down Allwinner A523 tablet from its stock software to a self-built LineageOS 23.2 / Android 16 TrebleDroid
GSI that clean-booted and passed the hardware and runtime checks described below.

The tablet began as a familiar white-label bargain: acceptable hardware, a recent-looking Android version, far too
many privileged OEM applications, inconsistent device identity, no published factory image, a locked boot chain,
and an update dialog that effectively held the user interface hostage. It ended as a fast, clean Android 16 tablet
that now serves as the operator's daily driver while retaining its stock kernel and vendor-side hardware components.

This is not presented as a universal flashing tutorial. It is a case study in careful preservation, Android boot
architecture, static reverse engineering, Project Treble, and methodical debugging. Another A523 tablet may look
identical and still have a different partition map, panel, touch controller, boot image, or fastbootd binary. Treat
every offset and hash in this document as evidence for this one firmware build—not as a magic constant.

## Contents

- [Scope, ethics, and risk](#scope-ethics-and-risk)
- [The target](#the-target)
- [Why this project started](#why-this-project-started)
- [Stage 1: investigating the forced OEM update](#stage-1-investigating-the-forced-oem-update)
- [Stage 2: preserving an otherwise unobtainable firmware](#stage-2-preserving-an-otherwise-unobtainable-firmware)
- [Stage 3: obtaining root through FEL and Secure Storage](#stage-3-obtaining-root-through-fel-and-secure-storage)
- [Stage 4: why root did not unlock fastbootd](#stage-4-why-root-did-not-unlock-fastbootd)
- [Stage 5: the eight-byte fastbootd patch](#stage-5-the-eight-byte-fastbootd-patch)
- [Stage 6: building a recovery path before experimenting](#stage-6-building-a-recovery-path-before-experimenting)
- [Stage 7: defeating the second lock (dm-verity)](#stage-7-defeating-the-second-lock-dm-verity)
- [Stage 8: making room inside dynamic super](#stage-8-making-room-inside-dynamic-super)
- [Stage 9: proving the GSI strategy](#stage-9-proving-the-gsi-strategy)
- [Stage 10: building our own LineageOS GSI](#stage-10-building-our-own-lineageos-gsi)
- [Stage 11: porting the patch stack to Android 16](#stage-11-porting-the-patch-stack-to-android-16)
- [Stage 12: the build failures that mattered](#stage-12-the-build-failures-that-mattered)
- [Stage 13: the smooth UI that took two minutes to react](#stage-13-the-smooth-ui-that-took-two-minutes-to-react)
- [Stage 14: restoring honest device identity](#stage-14-restoring-honest-device-identity)
- [Stage 15: reverse engineering Play restriction 9](#stage-15-reverse-engineering-play-restriction-9)
- [The finished tablet](#the-finished-tablet)
- [What is reusable and what is device-specific](#what-is-reusable-and-what-is-device-specific)
- [Guidance for similar projects](#guidance-for-similar-projects)
- [lineage-23.2-20260812-UNOFFICIAL-arm64_bgN-signed.img](#download-image)
- [Source repositories and integration model](#source-repositories-and-integration-model)
- [Public projects and references](#public-projects-and-references)
- [Closing note](#closing-note)

## Scope, ethics, and risk

All work described here was performed on hardware we own and firmware extracted from that hardware. The project does
not bypass FRP, an account lock, DRM, or another person's security. We did not attempt to forge a different certified
device identity. Where Android identity mattered, we restored the tablet's own factory values from its stock images.

The low-level steps can permanently damage a device if applied to the wrong image or offset. Preserve original
partitions, verify every read and write by hash, and keep a hardware recovery route. Do not disable Android Verified
Boot reflexively. Leaving the original vbmeta partitions untouched narrowed the boot-chain changes; actual
recoverability came from verified backups plus the validated recovery and FEL paths.

## The target

The device is sold as a FancyDay C10, built around the relatively niche Allwinner A523 (`sun55iw3p1`, also called
`Saturn`). The A523 is an interesting testbed precisely because it is cheap, modern enough to support Treble, and
obscure enough that normal enthusiast conveniences are mostly absent.

Our unit had:

- an Android 14 system layered over an Android-13-level vendor stack;
- a `5.15.123-android13-8` GKI-derived kernel;
- 64-bit ARM with 32-bit application compatibility;
- retrofit Virtual A/B and a dynamic `super` partition;
- eMMC storage;
- a Mali-G57 GPU and Allwinner CedarX video engine;
- AIC8800 Wi-Fi/Bluetooth;
- an eDP 1280×800 panel;
- a Silead `gslX680` touch controller;
- two cameras behind an old camera-provider interface;
- audio, sensors, lights, key management, and media supplied by stock Allwinner vendor binaries.

The external slot is microSD, not SIM. Our initial generic system nevertheless advertised cellular capabilities that
the hardware does not have—one of several small configuration mismatches that later mattered more than expected.

The tablet was inexpensive. The investigation was not.

## Why this project started

The original problem was not a desire to run a newer Android release. It was an unskippable OEM update dialog.

The dialog appeared above the interface, could not be dismissed, and made the tablet effectively useless unless the
user proceeded. It was not a normal Android system-update notification. It came from a privileged UID-1000 system
application with overlay permission, started at boot, downloaded its payload in the background, and displayed the
forced prompt afterward.

That was enough to ask a basic question: what, exactly, was this device trying to install?

The answer turned the project from debloating into a tour through Android networking, secure boot, bootloader state,
Project Treble, and a surprising amount of archaeology.

## Stage 1: investigating the forced OEM update

### Finding the real updater

The tablet contained two updater-like system applications. The obvious Softwinner updater was not responsible for
the blocking dialog. Window-manager state traced the overlay to `com.yhk.qeota`, a YHK/EFERCRO application running
as the Android system UID.

Its update flow was coercive by design. The server response explicitly set:

```text
hasUpdate = true
isForce = true
isAutoInstall = true
```

The check API also supplied the download URL dynamically, so searching the filesystem after the dialog appeared was
too late. The background download could resume across boots.

### The domains and endpoints

Dynamic interception and static APK analysis identified the following infrastructure:

| Host | Transport | Role |
|---|---|---|
| `idevice.efercro.com:10546` | TLS on non-standard port 10546 | Update check and telemetry |
| `backotadown.efercro.com` | HTTPS | OTA package download |
| `device.efercro.com` | Plain HTTP fallback | Device/update API |
| `devicetest.efercro.com` | Plain HTTP test path | Legacy/test endpoint |
| `api.bigbigcloud.cn` | Plain HTTP | Separate Softwinner DeviceHive REST channel |
| `ws.bigbigcloud.cn` | Plain WebSocket | Separate persistent DeviceHive command/telemetry channel |

The non-standard TLS port was the key detail. Initial transparent interception covered ports 80 and 443 and saw the
package download, but not the update decision. A high-frequency socket trace revealed the short-lived connection on
10546. Once the proxy and DNAT path accounted for that port, the entire protocol became visible.

### The signature that was not really a signature

The request's `sign` field was not an HMAC and did not use a device secret. It was:

```text
Base64(SHA1(field1 & field2 & ... & fieldN & field1))
```

The first field was repeated at the end. For the version check, the inputs were the package identifier, current
version, timestamp, and package identifier again. For download telemetry, the device identifier occupied the first
and last positions.

A small .NET client reproduced the check, download, and telemetry sequence. The live service accepted newly generated
requests, confirming that the scheme provided no meaningful request authenticity.

### The update package

The offered delta was roughly 347 MB and targeted nearly every important A/B partition, including:

- `init_boot`;
- `vendor_boot`;
- top-level and chained vbmeta partitions;
- `system`, `product`, and `vendor`;
- DTBO and DLKM partitions.

Applying it would overwrite Magisk in `init_boot`, restore stock boot components, and erase the working modified
boot state. The captured payload did not advertise Secure Storage itself as a target, but a future OTA or full
firmware package could change the bootloader-side policy that consumes it. We did not want to discover that after the
only convenient door had closed.

**Our recommendation is not to apply this OTA while relying on the documented Secure Storage method.** The update
channel's coercive UX, serial-based targeting, unkeyed SHA-1 request scheme, plaintext fallbacks, and broad partition
payload do not inspire confidence. An update may contain legitimate fixes, but it should be treated as an opaque
boot-chain replacement—not as a harmless monthly patch.

### Other OEM irregularities

Static analysis found no evidence that the forced OTA app harvested messages, contacts, location, or microphone data.
It was bad security engineering and coercive UX, not a classic mass-PII collector.

The separate `com.softwinner.update` package was more concerning: static analysis found a cleartext DeviceHive client
coded to send device identifiers, MAC-derived information, vendor, and device class to `bigbigcloud.cn`, with both
REST and a persistent WebSocket channel. A factory test application also shipped with broad camera, microphone, location,
contact, telephony, and reboot permissions. Its code looked like factory-line diagnostics, not remote spyware, but it
had no reason to remain on a consumer tablet.

One odd footnote: when the exact EFERCRO domains were compiled into our reverse-engineered .NET OTA client, Windows
Smart App Control blocked that binary while allowing our other unsigned .NET research tools. This is not proof that
the domains are malicious—reputation systems also block new and low-prevalence binaries—but it was a memorable bit of
ambient commentary from the operating system.

## Stage 2: preserving an otherwise unobtainable firmware

No usable factory image was publicly available for this exact tablet. Before writing anything, we treated the device
as an irreplaceable source specimen.

We recorded:

- the real GPT and every partition's size and offset;
- the original `init_boot` image;
- the proprietary Secure Storage region;
- the bootloader environment partition;
- stock boot, DTBO, vbmeta, vendor_boot, vendor, vendor_dlkm, ODM, system, and product content;
- vendor firmware, kernel modules, audio/media/camera configuration, SELinux policy, input configuration, and VINTF
  manifests;
- hashes for every critical image, with more than one copy.

This mattered immediately: the partition layout differed from the public reference tablet. Copying another board's
LBAs would have written to the wrong place. The first real operation must be reading the target device's own GPT.

The preservation work also revealed an important architectural fact: almost all hardware support lived in the stock
vendor side, not in the OEM system image. That made a GSI strategy realistic.

## Stage 3: obtaining root through FEL and Secure Storage

### Why normal Android routes failed

The Developer Options unlock toggle did not provide a usable bootloader unlock. Standard fastboot unlock commands
were missing, ignored, or volatile. The early boot chain—TOC0, BL31, OP-TEE, and U-Boot—was signed. TrustZone also
prevented the obvious approach of loading code into DRAM from a non-secure FEL context.

The public breakthrough was Chris Lennon's MIT-licensed
[`A523-root`](https://github.com/chrislennon/A523-root) project, built on
[`xfel`](https://github.com/xboot/xfel/). Its key insight is wonderfully Allwinner: do not fight the secure world in
DRAM. Run a tiny bare-metal eMMC driver entirely from the SRAM exposed through the SoC's immutable BootROM FEL mode.
The driver talks directly to the SMHC2 controller in PIO mode and provides raw eMMC reads and writes without modifying
the signed boot chain.

In broad terms, the process was:

1. identify the exact SoC, storage controller, active slot, and partition layout;
2. enter FEL and load the SRAM eMMC helper;
3. read the GPT and make verified backups;
4. dump the active `init_boot` partition;
5. patch that image with Magisk;
6. write it back and verify the read-back hash;
7. patch Allwinner Secure Storage so modified Android boot images are accepted.

FEL USB resets and re-enumeration are timing-sensitive. A native Linux environment with direct USB access proved far
more reliable than virtualized USB-over-IP. A real Linux machine, live environment, or VM with true controller
passthrough is the least surprising option.

### The Secure Storage change

The public A523-root tooling reads the proprietary Secure Storage block beginning at eMMC sector 12288, validates its
map and CRCs, and adds:

```text
fastboot_status_flag = unlocked
device_unlock        = unlock
```

On this unit, the region was 128 KiB and was backed up before modification. Sector 12288 is specific to the method and
this device family; anyone reproducing the work should validate their own map before writing.

The modified region allowed U-Boot to follow its unlocked/warn-and-continue path and boot a Magisk-patched
`init_boot`. We deliberately left vbmeta unchanged.

Root worked, but Android still reported `verifiedbootstate=green` and `vbmeta.device_state=locked`. That apparent
contradiction became the next blocker.

## Stage 4: why root did not unlock fastbootd

Rooting and booting a modified ramdisk did not make userspace fastboot writable. In fastbootd:

```text
unlocked: no
Flashing is not allowed on locked devices
Command not available on locked devices
```

The Secure Storage flags were real, but fastbootd did not consult them. Static analysis showed that this firmware's
fastbootd was effectively stock AOSP Android 13 code. Every mutating operation called one function:

```cpp
bool GetDeviceLockStatus() {
    return GetProperty("ro.boot.verifiedbootstate", "") != "orange";
}
```

U-Boot kept reporting `green`, so fastbootd kept saying no.

We considered changing the boot property. A bootconfig override was rejected because duplicate keys can invalidate the
entire bootconfig on the 5.15 kernel. A command-line override was possible but would alter system-wide attestation
state. A local binary change affected only recovery/fastbootd and was easier to reason about.

## Stage 5: the eight-byte fastbootd patch

For our stock `vendor_boot` partition:

```text
Size:    33,554,432 bytes
SHA-256: 95dc7e6cb526f3084fe83e8ce9cf5ca06136729513395279bf9f87564150f5dd
```

The `fastbootd` binary inside its LZ4 vendor ramdisk had `GetDeviceLockStatus()` at file offset `0x0a4d1c`.
We replaced the function prologue with `return false`:

| | Bytes | AArch64 meaning |
|---|---|---|
| Original | `ff 03 02 d1 fd 7b 06 a9` | Stack-frame prologue |
| Patched | `00 00 80 52 c0 03 5f d6` | `mov w0,#0; ret` |

After rebuilding and writing `vendor_boot`, fastbootd reported `unlocked: yes`, and flash, erase, logical-partition,
resize, and set-active operations worked.

**Do not reuse this offset blindly.** It is valid only for a binary matching the partition hash above. On another
firmware, locate the function through its references to `ro.boot.verifiedbootstate`, `orange`, or the fastboot lock
messages, and require an exact original-byte match before patching. If the eight expected bytes are not present,
stop. A wrong eight-byte patch is still eight bytes too many.

The change is local to fastbootd. Normal Android boot does not execute this binary, and `verifiedbootstate` remains
unchanged.

## Stage 6: building a recovery path before experimenting

A writable fastbootd is useful only if a broken GSI cannot prevent reaching it.

Read-only reverse engineering of the signed Allwinner boot chain found:

- TOC0 begins at raw eMMC sector 16;
- the real U-Boot lives in signed boot packages before the GPT;
- the GPT `bootloader` partition mostly contains resource files and warning graphics, not the U-Boot code;
- U-Boot samples the low-resolution ADC keys on every boot;
- it understands recovery, limited bootloader-fastboot, FEL, BCB reboot reasons, and Android A/B metadata;
- Android's `reboot fastboot` route is implemented as recovery plus a `--fastboot` BCB argument.

The simplest recovery layer required no new patch: a hardware key combination reaches recovery, and recovery offers
"Enter fastboot," which launches the already patched fastbootd entirely from `vendor_boot`. This path does not depend
on the GSI system partition.

We also studied automatic boot-attempt and first-stage-init failsafes. The tablet's A/B metadata can decrement boot
attempts, but a slot already marked successful can bootloop forever without automatic fallback. A theoretical
first-stage counter could redirect repeated failures to fastbootd, but the validated hardware recovery path was
simpler and sufficient. FEL remains the final recovery floor because the BootROM itself cannot be overwritten.

## Stage 7: defeating the second lock (dm-verity)

Fastboot could now write a GSI, but the kernel still had to mount it.

The first-stage fstab lived in the `vendor_boot` ramdisk. Its two `system` entries requested AVB/dm-verity using the
stock vbmeta chain. A different system image necessarily had a different hashtree root. Under the tablet's reported
`green`, `locked`, and `enforcing` state, the first verified read would trigger `restart_on_corruption` and bootloop
before animation.

The minimal fix was not a replacement vbmeta. We removed the AVB arguments from the two `system` fstab entries, while
keeping both EROFS and ext4 mount variants. With no `avb` option on that fstab entry, first-stage `fs_mgr` mounts the
logical system partition directly.

This preserved the original vbmeta partitions and avoided turning a system-image experiment into a boot-chain
experiment.

## Stage 8: making room inside dynamic super

The physical `super` partition was only about 3.76 GB. The stock layout devoted substantial space to a separate
`product` logical partition, while TrebleDroid GSIs carry product content inside the system image.

We deleted `product_a` to make room and resized `system_a`. That exposed another easily overlooked blocker: the
first-stage fstab still contained a mandatory logical `product` entry. An absent mandatory logical partition is fatal,
so the product line had to be removed from the same fstab.

At this point the persistent device-side foundation was:

- stock kernel, boot, vendor, vendor_dlkm, ODM, firmware, and proprietary HALs;
- Magisk in `init_boot`;
- unlocked fastbootd and verity-adjusted fstab in `vendor_boot`;
- original vbmeta partitions;
- a larger writable `system_a` and no standalone `product_a`.

System-image iteration was now measured in minutes instead of FEL's very patient PIO speed.

## Stage 9: proving the GSI strategy

### Pure AOSP first

We built a pure AOSP Android 13 `aosp_arm64` GSI to validate the source and build pipeline. It was useful as a control,
but pure AOSP lacked the compatibility layer needed for a comfortable old-vendor/new-system pairing.

### AndyYan and TrebleDroid

The decisive reference was an AndyYan LineageOS 20 TrebleDroid GSI. It combined three layers:

1. **The stock vendor stack**, which did the real hardware work.
2. **A generic LineageOS system**, communicating through Project Treble interfaces.
3. **TrebleDroid/phh compatibility glue**: scripts, runtime properties, overlays, and TrebleApp switches for vendor
   quirks.

The result booted with correct orientation and density immediately because those display values came from stock
vendor properties. Touch, Wi-Fi, Bluetooth, audio, hardware composition, and 4K playback also worked; the codec path
was not independently verified.

This established the project's core principle: **reuse the closed hardware stack; reverse engineer only the glue.**
Reimplementing Mali, CedarX, camera ISP tuning, or OEMCrypto would be a heroic way to make the tablet worse.

## Stage 10: building our own LineageOS GSI

Using Andy Yan's public
[`lineage_build_unified`](https://github.com/AndyCGYan/lineage_build_unified) and
[`lineage_patches_unified`](https://github.com/AndyCGYan/lineage_patches_unified), plus TrebleDroid and
MindTheGapps, we built our own LineageOS 20 Android 13 image.

That exercise added several practical lessons:

- a GSI target still needs a large, reproducible Android source environment;
- patch order matters because the build scripts apply repository-local mail patches alphabetically;
- signed GApps builds require a surprisingly complete APK/APEX key set;
- a system-only GSI correctly emits warnings about absent kernel and boot images;
- a development CA can be built into the system trust store for owner-controlled protocol research, but should be
  replaced with one's own CA—or omitted entirely—in a general-purpose release;
- root belongs in `init_boot`, not in the GSI, so system images remain interchangeable.

After Android 13, we built and validated LineageOS 21 / Android 14. That gave us a known-good source-built baseline
before attempting Android 16.

## Stage 11: porting the patch stack to Android 16

There was no ready-made AndyYan LineageOS 23 branch for this target, so we ported the Android 14 TrebleDroid/Lineage
patch stack forward to LineageOS 23.2.

The original review contained 306 patch rows across many Android repositories. AI-assisted research and verification
agents helped trace moved files, changed APIs, obsolete behavior, and cross-repository dependencies. This accelerated
the review, but the project remained evidence-driven: every retained, rebased, or omitted patch needed source-level
justification and a clean application result.

The review ended with:

```text
265 retained baseline patches
41 verified omissions
0 unresolved rows
```

Thirty additional integration and runtime patches were added during real builds and hardware testing, for a final
295-patch tree.

We intentionally used an **additive** TrebleDroid source layer. Canonical TrebleDroid's broad replacement manifest
would swap LineageOS platform repositories for AOSP-based forks and discard the Lineage behavior we were porting.
Instead, Lineage platform repositories remained in place and received the reviewed compatibility patches; phh,
overlays, VNDK support, Magisk utilities, and MindTheGapps were added beside them.

## Stage 12: the build failures that mattered

Most of the port was not dramatic. It was dozens of small incompatibilities that each prevented a complete image.
Among the fixes:

- removed an obsolete LGE vibrator HIDL definition and its helper after Android 16 removed that interface;
- replaced the removed AOSP Calendar module with LineageOS Etar;
- marked retained Samsung framework HIDL interfaces as generic so GSI image checks accepted them;
- removed stale package references for overlay helpers, `healthd`, and the old VR compositor;
- deleted a dead GLES-loader state that Android 16 promoted to a `-Werror` failure;
- adapted alternate-camera opening to the Android 16 API;
- updated restored telephony logging symbols;
- completed a sequence of restored `CommandsInterface` and `FakeRil` stubs—the tablet has no modem, but Java
  interfaces still have to compile completely;
- repaired SystemUI notification-shelf fallout from an old logger revert;
- replaced a private bionic alignment macro in Magisk `resetprop` with a public compiler builtin;
- migrated TrebleApp's namespace from the deprecated manifest package attribute to Gradle;
- taught the build-property generator to carry an exact display ID without appending build-key text;
- extended the Android 16 signing key set to all required APEX containers and payloads.

The lesson is unglamorous and important: compatibility projects often succeed through thirty tiny correct decisions,
not one spectacular hack.

## Stage 13: the smooth UI that took two minutes to react

The first fully booting Android 16 image looked deceptively good. Animations were smooth. Touch responded. Yet volume
buttons took minutes to show a slider, Setup Wizard crashed or went black, and Android reported that the system process
was not responding.

That combination ruled out the obvious graphics and touch theories.

The ANR trace showed `system_server` holding the ActivityManager lock while waiting for `lmkd`. The daemon was exiting
every five seconds. The stock vendor property requested Android's old low-memory-killer strategy, but the tablet
mounted the memory controller through cgroup v2. Android 16 explicitly rejected the old cgroup-v1-only strategy, then
its fallback also failed.

A system property override was not enough because the immutable vendor property won during boot. The durable fix was
inside Android 16 `lmkd`: when the memory hierarchy is cgroup v2, force the compatible PSI/new strategy rather than
honoring a stale request that can never work.

After that change:

- `lmkd` stayed running;
- its control socket remained present;
- ActivityManager stopped retrying;
- the ANRs disappeared;
- the tablet became immediately responsive.

This was the project's best reminder that smooth graphics do not imply a healthy operating system.

We also made LineageOS rooted debugging default-on for this owner-development build and applied that policy during
boot, so ADB arrives as root without an extra command. That is useful for a research image and inappropriate for a
normal consumer release.

## Stage 14: restoring honest device identity

Generic GSIs report generic product information by design. Our first images exposed values such as `mainline`,
`unknown`, and the generic GSI product name in merged product/system_ext properties. Android's product-property
precedence allowed those values to outrank otherwise-correct stock vendor identity.

We recovered the shipping Play-facing fingerprint and build description, plus the retail FancyDay/C10US/C10
identity, from the stock images. Stock product, system_ext, vendor, and ODM used the retail identity, while stock
system still contained Allwinner reference-board `ro.product.system.*` values.

For the GSI we deliberately selected the coherent retail values across merged partitions:

- brand/manufacturer: `FancyDay`;
- product/name: `C10US`;
- model/device: `C10`;
- characteristics: `tablet`;
- factory Play-facing fingerprint and build description;
- retail display ID: `863C_C10_20240619`.

We kept the Android 16 release, SDK, and security-patch values truthful and retained the vendor-reported first-API
properties and vendor fingerprints. We did not substitute another manufacturer's identity or add attestation
overrides.

The build wrapper gained hard assertions so a future regression to generic `google/unknown/tdgsi` values fails before
signing.

## Stage 15: reverse engineering Play restriction 9

After identity restoration, Chrome and TikTok were still hidden by Play Store while Earth and YouTube were available.
This showed that identity restoration alone was insufficient. Play reported the device certified, so this was not a
simple certification failure.

Static analysis of the installed Play Store mapped log restriction `9` to `UNAVAILABLE_DEVICE_HARDWARE`. A fresh,
cache-cleared TLS capture showed the server's modern response contained only:

```text
UNAVAILABLE_BECAUSE_DEVICE_INCOMPATIBLE
```

There was no hidden field describing the failed capability. The compatibility decision was server-side and keyed to
the device profile registered through GMS check-in.

We captured and decoded GMS check-ins from the C10 and from a known-good Chrome-compatible tablet on the same account.
The result was concrete:

- GMS serialized `pm list features` verbatim;
- the C10 uploaded eight false cellular hardware features;
- the same GSM XML also implied `android.software.telecom`;
- the C10 omitted the location, GPS, and network-location declarations even though location/GPS worked in Google Earth;
- ABI sets matched the known-good tablet;
- the C10's GLES/GL extension set was a superset;
- screen class and smallest width were coherent;
- the same Android ID could submit an updated profile successfully.

A reversible live overlay applied the exact proposed feature correction before PackageManager started. We triggered a
same-ID check-in, cleared only Play Store's seven-day details cache, and reopened Chrome. Chrome became normally
visible, downloadable, and installable.

The final source patches therefore:

- remove the two cellular feature XMLs on this Wi-Fi-only model;
- add canonical Android declarations for location/network and GPS.

No foreign identity spoofing or Play client modification was required. In this same-ID experiment, correcting the
aggregate feature profile was sufficient. The test did not isolate which individual declaration affected the catalog
verdict, so no single XML or identity property is claimed as the cause. The bad profile was still wearing a trench
coat; we just stopped pretending to know which pocket held the problem.

## The finished tablet

The final result is a self-built LineageOS 23.2 / Android 16 TrebleDroid GSI signed with project-generated release
keys—not official LineageOS or Google keys. It runs with the retained stock kernel, vendor, vendor_dlkm, ODM,
firmware, and proprietary HALs; `init_boot` and `vendor_boot` were intentionally modified as described above.

Validated behavior:

| Area | Result |
|---|---|
| Boot and responsiveness | Clean boot verified; lmkd healthy; prior system-server ANR/retry storm absent |
| Display/composition | HWC active; smooth UI with correct rotation and density |
| Graphics APIs | Mali rendering works; Vulkan/GLES conformance was not acceptance-tested |
| Touch | Responsive and correctly aligned |
| Wi-Fi | Associated and provided Internet and Play Store access during testing |
| Bluetooth | Working |
| Audio | Speaker playback working |
| Cameras | Both cameras take photos and record video |
| Sensors | Accelerometer and automatic rotation working |
| GPS | Accurate location confirmed in Google Earth |
| Video | 4K30 playback smooth; 4K60 lags somewhat; codec path not independently confirmed |
| Storage/microSD | Slot identified and stock vendor path retained; removable-media I/O not acceptance-tested |
| Root/development | Magisk persists in init_boot; research build offers rooted ADB |
| Play Store | Device certified; Chrome and tested Google applications install normally |

The tablet has since entered service as the kid's daily driver, where it has passed the most demanding benchmark of
all: no complaints so far. Every original user-facing subsystem exercised during the project works, and no
ordinary-use hardware regression has been observed. The experience is dramatically better than
stock: faster, cleaner, more predictable, and no longer governed by a forced update overlay. The obscure bargain-bin
tablet has become a surprisingly pleasant Android 16 machine—which is a sentence its original software did not seem
eager to permit.

This is a field result, not a formal certification run. Not acceptance-tested: Vulkan/GLES conformance, Widevine/DRM,
KeyMint/Gatekeeper behavior, face unlock, auto-brightness/vendor display extras, removable-media I/O, codec-path
verification, CTS/VTS, and long-duration soak stability. The slight 4K60 performance limit is not surprising for this
hardware; 4K30 is the practical success target.

## What is reusable and what is device-specific

### Broadly reusable ideas

These lessons apply to many Treble devices:

- preserve before modifying;
- keep the stock vendor/kernel stack when proprietary hardware works;
- use a GSI to separate system experimentation from vendor driver bring-up;
- distinguish bootloader unlock, fastbootd policy, and kernel dm-verity—they are different gates;
- trace first-stage fstab behavior rather than blindly flashing disabled vbmeta;
- verify generated product identity across every merged partition;
- use unsigned images for fast iteration and sign only acceptance builds;
- treat a successful boot as the beginning of runtime validation, not the end;
- use server-request captures and device-profile diffs instead of guessing at Play compatibility.

### Specific to this firmware or SoC

The following must be omitted, relocated, or re-derived on another device:

1. **Secure Storage sector and items.** Sector 12288 and the two item names come from the public A523-root method and
   matched this unit. Validate the target's own Secure Storage map and CRCs first.
2. **Fastbootd offset and bytes.** Offset `0x0a4d1c` is specific to the stock `vendor_boot` SHA-256 shown earlier.
   Locate the function independently on every build.
3. **The `sun55iw3p1` first-stage fstab edits.** Other boards may store fstab in DT, init_boot, another ramdisk path,
   or use different AVB chains and logical partitions.
4. **Deleting `product_a`.** This was necessary because this tablet's super partition was small and our GSI merged
   product content. A larger or non-retrofit device may not need it.
5. **C10 identity overrides.** FancyDay/C10US/C10 and the retail display ID belong only to this product. Restore the
   target device's own identity—never copy ours.
6. **Cellular feature removal.** This C10 is Wi-Fi-only. Another A523 SKU may contain a modem and must retain its
   telephony declarations.
7. **Location feature additions.** Add only features proven by hardware tests.
8. **The lmkd cgroup-v2 fallback.** It is useful where a stale vendor forces the old strategy on a cgroup-v2 kernel;
   it is unnecessary when vendor properties are already correct.
9. **Rooted ADB default.** This is intentionally development-oriented and should normally be omitted.
10. **The research CA.** Use your own CA for private protocol work and remove it from images intended for others.
11. **Camera and vendor compatibility patches.** Some Android 16 API adaptations are broadly useful; OEM-specific
    Samsung, LGE, Qualcomm, Xiaomi, and telephony compatibility patches may be irrelevant or harmful on a trimmed
    single-device build.

A future device-specific ROM would ideally replace the large unified compatibility surface with a small tree that
selects only the A523 blobs, firmware, overlays, and policy actually used by the board. The GSI route remains the
fastest and most maintainable solution today.

## Guidance for similar projects

This is deliberately not a copy-and-paste flashing guide. A responsible replication effort should answer these
questions in order:

1. **What is the real hardware?** Confirm SoC, storage type, active slot, launch API, vendor VNDK, kernel branch,
   dynamic-partition layout, panel, touch, Wi-Fi, and camera hardware.
2. **Can the device always be recovered?** Establish BootROM/FEL or another immutable recovery path before writing.
3. **Are there complete backups?** Preserve GPT, Secure Storage/environment, boot images, vbmeta, super, vendor,
   system, and product; verify hashes and keep offline copies.
4. **What does each lock actually protect?** Separate signed early boot, Secure Storage policy, Android AVB,
   fastbootd's userspace checks, and dm-verity.
5. **Can stock vendor be reused?** On Treble devices, this is usually preferable to replacing proprietary drivers.
6. **What is the smallest first GSI?** Start near the vendor's Android generation before jumping forward.
7. **What broke at build time versus runtime?** Compile errors, product-graph errors, early-boot failures, and
   post-boot ANRs require different evidence.
8. **Is a reported compatibility problem local or server-side?** Capture and compare the exact profile before
   changing random properties.
9. **Are changes reproducible?** Keep every source modification as an ordered patch, record hashes, and enforce
   acceptance properties before signing.

A useful AI assistant can accelerate source searches, API migration, decompilation triage, and cross-repository
comparison. It should not replace byte-match gates, hashes, backups, source citations, or hardware tests. Ask agents
to distinguish proof from inference and to state when an answer is device-specific.

<a id="download-image"></a>
## lineage-23.2-20260812-UNOFFICIAL-arm64_bgN-signed.img

The final signed image is distributed as a gzip archive:

[Download `lineage-23.2-20260812-UNOFFICIAL-arm64_bgN-signed.img.gz` from SourceForge](https://downloads.sourceforge.net/project/a523-trebledroid-lineageos-23/lineage-23.2-20260812-UNOFFICIAL-arm64_bgN-signed.img.gz)

| Artifact | Size | SHA-256 |
|---|---:|---|
| Downloaded `.img.gz` | 1,509,033,632 bytes | `f33754dec15d9679a29cec8439aed48967ed25a5bfe60193fdc1c3fc325f7de6` |
| Decompressed `.img` | 3,225,112,576 bytes | `1ff235092605a79ac4b44e8eb8ca2cf53260c2352424cabbfbf4c43f2f4460c2` |

### Before flashing: check dm-verity

> **Warning:** flashing this image while the stock first-stage `system` entry still enables dm-verity is a
> guaranteed bootloop. The GSI cannot match the stock `vbmeta_system` hashtree digest.

Follow [Checking and disabling dm-verity on the FancyDay C10](DM_VERITY.md) before flashing. The guide begins with a
read-only runtime and `vendor_boot` inspection, provides an explicit write/no-write decision table, and keeps the
persistent modification commands in a separately gated section.

Do **not** work around this by flashing a verification-disabled vbmeta. Keep the original vbmeta partitions untouched.

## Source repositories and integration model

The Android 16 source contribution is split across two matching `lineage-23-td` branches:

- [`MeisterLone/lineage_build_unified`](https://github.com/MeisterLone/lineage_build_unified/tree/lineage-23-td)
  is the **orchestrator**. It installs the additive TrebleDroid manifest, synchronizes the required projects, resets
  target repositories, applies patch groups in order, generates the phh/Lineage products, builds TrebleApp and the
  GSI, runs the C10 identity/hardware gates, and optionally signs the target-files package.
- [`MeisterLone/lineage_patches_unified`](https://github.com/MeisterLone/lineage_patches_unified/tree/lineage-23-td)
  is the **payload**. It contains the reviewed Android 16 compatibility patches and the later build/runtime fixes.
  The build wrapper does not duplicate those changes; it reads and applies this repository.

Clone both repositories side by side in the root of a LineageOS 23.2 source tree, using their expected directory
names and the same branch:

```bash
git clone -b lineage-23-td \
  https://github.com/MeisterLone/lineage_build_unified.git lineage_build_unified
git clone -b lineage-23-td \
  https://github.com/MeisterLone/lineage_patches_unified.git lineage_patches_unified
```

The wrapper verifies that both checkouts are on `lineage-23-td` and then applies the patch repository in this order:

```text
patches_treble_prerequisite
patches_treble_td
patches_platform
patches_treble
patches_platform_personal   # only when explicitly requested
```

For the C10 GApps build, the paired entry point is:

```bash
bash lineage_build_unified/buildbot_unified.sh treble 64GN
```

`64GN` selects the ARM64 A/B GSI with MindTheGapps and no bundled phh superuser. Root still comes from Magisk in the
device's retained `init_boot`. A signing keyset enables the wrapper's target-files signing path; without one, use an
unsigned image for development and reserve signing for acceptance builds.

The additive manifest is intentional. Do not combine these branches with TrebleDroid's broad `replace.xml`: that
would replace LineageOS platform repositories with AOSP-oriented TrebleDroid forks and bypass the very patch port this
pair exists to build.

## Public projects and references

This work depended on public research and open-source projects. Credit belongs upstream.

### Root, FEL, and Allwinner

- [chrislennon/A523-root](https://github.com/chrislennon/A523-root) — the public A523/T527 FEL, SRAM eMMC,
  Magisk, and Secure Storage method that made this project possible.
- [xboot/xfel](https://github.com/xboot/xfel) — FEL tooling with A523 support.
- [Magisk](https://github.com/topjohnwu/Magisk) — `init_boot` patching and persistent root.
- [linux-sunxi A523/T527 wiki](https://linux-sunxi.org/T527) — community SoC information.
- [SyterKit](https://github.com/YuzukiHD/SyterKit) — useful A523/T527 bare-metal reference work.
- [Allwinner A523 discussion and prior art on XDA](https://xdaforums.com/t/rom-pritom-tab11-tablet-allwinner-a523.4783482/)
  — early public GSI evidence for this SoC family.

### Project Treble and GSIs

- [Android Generic System Image documentation](https://source.android.com/docs/core/tests/vts/gsi)
- [Android Dynamic System Updates documentation](https://source.android.com/docs/core/ota/dynamic-system-updates)
- [TrebleDroid/treble_experimentations](https://github.com/TrebleDroid/treble_experimentations)
- [phhusson/treble_experimentations](https://github.com/phhusson/treble_experimentations) — historical PHH-Treble
  work; archived in favor of TrebleDroid.
- [AndyCGYan/lineage_build_unified](https://github.com/AndyCGYan/lineage_build_unified)
- [AndyCGYan/lineage_patches_unified](https://github.com/AndyCGYan/lineage_patches_unified)
- [LineageOS](https://github.com/LineageOS)
- [MindTheGapps vendor_gapps](https://gitlab.com/MindTheGapps/vendor_gapps)

### Android source references

- [Android Open Source Project documentation](https://source.android.com/docs)
- [AOSP source browser](https://cs.android.com/android/platform/superproject/main)
- [Android build documentation](https://source.android.com/docs/setup/build/building)

## Closing note

This project began because a cheap tablet offered one button — "Update now" — and no way to say no. So we built the
"not now" it refused to give us. Then we made it permanent.

The result is not just Android 16 on an A523. It is a chain of understandable, reversible decisions: observe first,
preserve everything, identify each independent gate, reuse what already works, patch only what evidence justifies,
and verify on real hardware.

It was occasionally tedious, often surprising, and genuinely fun — which is about the best outcome a hobby research
project can hope for, aside from the tablet that provoked it all becoming a machine we're glad to keep.
