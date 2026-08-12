# Checking and disabling dm-verity on the FancyDay C10

This document covers one specific boot blocker encountered on the FancyDay C10 (`sun55iw3p1`): the stock
first-stage fstab enables dm-verity for `system`, but a GSI cannot match the stock `vbmeta_system` hashtree digest.
Flashing a different system image without addressing that mismatch causes a bootloop before Android finishes early
boot.

The procedure is deliberately split into two independent parts:

1. **Read-only diagnosis** — inspect the running device and a dumped copy of `vendor_boot`. This part does not write
   to any boot partition.
2. **Persistent modification** — edit and write `vendor_boot` only if the read-only evidence proves it is necessary.

Do not proceed to Part II merely because the device uses AVB somewhere. The deciding question is whether the
first-stage `system` fstab entry enables dm-verity.

> **Scope:** These paths and image details are verified on the FancyDay C10 firmware used by this project. Other
> A523 boards may store their first-stage fstab elsewhere or use a different vendor ramdisk layout. Stop if the
> expected files are absent; do not improvise by flashing disabled vbmeta.

## Requirements

- A Linux host with `adb`, Python 3, `lz4`, GNU `cpio`, and Git.
- Rooted adbd (`adb root`). The pasteable commands below assume `adb shell` is already uid 0.
- A known recovery route such as FEL or a working recovery/fastbootd path.
- AOSP's [`mkbootimg`](https://android.googlesource.com/platform/system/tools/mkbootimg) tools.

On Debian or Ubuntu, install the host tools and obtain `unpack_bootimg.py`/`mkbootimg.py` as follows. Other Linux
distributions should install the equivalent packages through their own package manager.

```bash
sudo apt update
sudo apt install -y adb git python3 lz4 cpio

if [ -e mkbootimg ] && [ ! -d mkbootimg/.git ]; then
  echo 'ERROR: ./mkbootimg exists but is not the expected Git checkout'
  exit 1
fi

test -d mkbootimg/.git || \
  git clone https://android.googlesource.com/platform/system/tools/mkbootimg
```

If only an interactive `su` implementation is available, adapt and test each on-device command separately. In
particular, do not stream a binary partition through an untested `su` wrapper because prompts or diagnostics can
corrupt stdout.

---

# Part I — Read-only, non-invasive diagnosis

Everything in Part I reads device state or copies data to the host. It does **not** write to `vendor_boot`, vbmeta,
or any other boot partition.

## 1. Confirm the target, root, and active slot

List connected devices first:

```bash
adb devices -l
```

If more than one device is listed, select the intended tablet before running any command below:

```bash
export ANDROID_SERIAL='<serial-or-ip:port-from-adb-devices>'
```

Do not continue while the target is ambiguous. Then confirm rooted adbd and identify the active slot:

```bash
adb root
adb wait-for-device
adb shell id

SLOT=$(adb shell getprop ro.boot.slot_suffix | tr -d '\r')
case "$SLOT" in
  _a|_b) ;;
  *) echo "ERROR: unexpected slot suffix: '$SLOT'"; exit 1 ;;
esac
printf 'Active slot: %s\n' "$SLOT"
```

The explicit gate prevents an empty slot value from silently turning `vendor_boot${SLOT}` into the wrong partition
name.

## 2. Inspect the active system device-mapper table

```bash
adb shell 'command -v dmctl'
adb shell "dmctl table system${SLOT}"
```

If the expected table name is absent, list the available mappings before making a decision:

```bash
adb shell 'dmctl list devices'
```

Example without dm-verity:

```text
0-2043952: linear, ...
2043952-5077040: linear, ...
```

Decision:

| Output | Meaning | Action |
|---|---|---|
| Every target is `linear` | The currently booted `system` mapping is not using dm-verity | Continue with the offline fstab inspection for confirmation; do not modify anything yet |
| Any target is `verity` | The active `system` mapping is protected by dm-verity | The downloaded GSI will not match the stock digest; continue to the offline fstab inspection |
| `dmctl` fails or the device name differs | Runtime result is inconclusive | Do not write anything; rely on the offline fstab inspection below |

The runtime table is useful evidence, but the first-stage fstab is the authoritative configuration that will be used
on the next boot.

## 3. Read the active vendor_boot to the host

The following streams the partition directly to a file on the Linux host. It does not create a temporary image on the
device or modify the partition.

```bash
PART="/dev/block/by-name/vendor_boot${SLOT}"
adb shell "test -b '$PART'"

PART_BYTES=$(adb shell "blockdev --getsize64 '$PART'" | tr -d '\r')
case "$PART_BYTES" in
  ''|*[!0-9]*) echo "ERROR: invalid partition size: '$PART_BYTES'"; exit 1 ;;
esac
test "$PART_BYTES" -gt 0

test ! -e vendor_boot.stock.img
adb exec-out "dd if='$PART' bs=4M 2>/dev/null" > vendor_boot.stock.img

DUMP_BYTES=$(stat -c %s vendor_boot.stock.img)
printf 'Partition bytes: %s\n' "$PART_BYTES"
printf 'Dumped bytes:    %s\n' "$DUMP_BYTES"
test "$DUMP_BYTES" = "$PART_BYTES"
sha256sum vendor_boot.stock.img
```

For provenance, the original C10 `vendor_boot` used in this project was:

```text
Size:    33,554,432 bytes
SHA-256: 95dc7e6cb526f3084fe83e8ce9cf5ca06136729513395279bf9f87564150f5dd
```

The hash is a reference, not a patch gate. This dm-verity procedure uses no fixed partition or byte offsets. A
different hash is acceptable if the image unpacks correctly and its own first-stage fstab is inspected.

## 4. Unpack vendor_boot without modifying it

`UNPACK` and `MKBOOT` below are variables pointing to the two Python tools in the AOSP `mkbootimg` repository cloned
under Requirements. This assumes `mkbootimg/` is in the current directory.

```bash
MKBOOTIMG_DIR=$(realpath ./mkbootimg)
UNPACK="$MKBOOTIMG_DIR/unpack_bootimg.py"
MKBOOT="$MKBOOTIMG_DIR/mkbootimg.py"
IN="$PWD/vendor_boot.stock.img"
WORK=$(mktemp -d -p "$PWD" vendor_boot_check.XXXXXX)
printf 'Working directory: %s\n' "$WORK"

test -f "$UNPACK"
test -f "$MKBOOT"
test -f "$IN"

mkdir -p "$WORK/out" "$WORK/root"
python3 "$UNPACK" --boot_img "$IN" --out "$WORK/out"
```

`WORK` now contains the absolute path of the newly created temporary directory on the host. All unpacked files,
modified files, rebuilt archives, and verification output in the remaining commands are kept beneath that directory.
Keep using the same terminal so the variable remains defined. If the shell is closed, rerun this step to create and
populate a new working directory rather than guessing its generated name.

The C10 vendor ramdisk is an LZ4-legacy fragment named `vendor_ramdisk00`:

```bash
test -f "$WORK/out/vendor_ramdisk00"
lz4 -d -f "$WORK/out/vendor_ramdisk00" "$WORK/ramdisk.cpio"

(
  cd "$WORK/root"
  cpio -idm --no-absolute-filenames < "$WORK/ramdisk.cpio"
)
```

Locate and inspect the first-stage fstab:

```bash
FSTAB="$WORK/root/first_stage_ramdisk/fstab.sun55iw3p1"
test -f "$FSTAB"
grep -nE '^[[:space:]]*(system|product)[[:space:]]' "$FSTAB"
```

The problematic C10 entries looked like this in principle:

```text
system  /system  erofs  ...  first_stage_mount,logical,slotselect,avb=vbmeta,avb_keys=/avb
system  /system  ext4   ...  first_stage_mount,logical,slotselect,avb=vbmeta,avb_keys=/avb
```

Check separately whether the active logical product partition exists. This check is read-only:

```bash
if adb shell "test -b '/dev/block/mapper/product${SLOT}'"; then
  echo "product${SLOT} exists — keep the product fstab entry"
else
  echo "product${SLOT} is absent — remove the product fstab entry only if it was deliberately deleted"
fi
```

An absent mapper by itself does not prove that deleting the fstab line is safe; it is a prompt to confirm how that
firmware lays out product content.

## 5. Make the write/no-write decision

Use the fstab output—not the device model name—as the final decision gate.

### Do not modify vendor_boot when

- neither `system` line contains `avb=` or `avb_keys=`; and
- the runtime `system` device-mapper table contains only `linear` targets.

In that state, this particular dm-verity fix is unnecessary.

### Modification is required before flashing this GSI when

- either first-stage `system` line contains `avb=vbmeta`, `avb_keys=/avb`, or equivalent dm-verity configuration.

The GSI's ext4 data and hashtree cannot match the stock `vbmeta_system` root digest. On this firmware, leaving those
flags enabled causes a guaranteed early bootloop.

### Stop and investigate instead of writing when

- `vendor_boot` does not unpack normally;
- `vendor_ramdisk00` is absent;
- `first_stage_ramdisk/fstab.sun55iw3p1` is absent;
- the fstab uses a different AVB arrangement than the two known C10 `system` entries; or
- there is no tested recovery route.

A different layout is not permission to copy C10 paths blindly.

---

# Part II — Persistent vendor_boot modification

> **Stop:** Part II writes an early boot partition. Continue only if Part I showed AVB/dm-verity flags on the
> first-stage `system` entries, all backups are verified, and a recovery route is available.

The following commands assume the variables and extracted tree from Part I still exist. Run the cpio extraction and
repacking operations as root so numeric ownership and modes are preserved.

## 1. Preserve a second backup

```bash
test -f vendor_boot.stock.img
test ! -e vendor_boot.stock.backup.img
cp -p vendor_boot.stock.img vendor_boot.stock.backup.img
sha256sum vendor_boot.stock.img vendor_boot.stock.backup.img
cmp vendor_boot.stock.img vendor_boot.stock.backup.img
```

Store the original image somewhere other than the device before proceeding.

## 2. Record the original archive order and re-extract as root

First verify that the Part I variables still point to real files, then enter a root shell with those paths passed
explicitly:

```bash
test -n "${WORK:-}"
test -d "$WORK"
test -f "$WORK/ramdisk.cpio"
test -f "$UNPACK"
test -f "$MKBOOT"
test -f "$IN"

sudo env WORK="$WORK" UNPACK="$UNPACK" MKBOOT="$MKBOOT" IN="$IN" \
  bash --noprofile --norc
```

The prompt is now a root Bash shell. Create a new extraction directory rather than deleting or reusing the read-only
Part I tree:

```bash
ROOT_WRITE="$WORK/root-write"
test ! -e "$ROOT_WRITE"
mkdir -p "$ROOT_WRITE"

cpio -it < "$WORK/ramdisk.cpio" > "$WORK/ramdisk.list"

(
  cd "$ROOT_WRITE"
  cpio -idm --no-absolute-filenames < "$WORK/ramdisk.cpio"
)

FSTAB="$ROOT_WRITE/first_stage_ramdisk/fstab.sun55iw3p1"
test -f "$FSTAB"
cp "$FSTAB" "$WORK/fstab.before"
```

This avoids a recursive cleanup command entirely. If `root-write` already exists, the gate stops instead of mixing
files from an earlier run.

## 3. Remove dm-verity flags from both system entries

The verified C10 fstab has exactly two `system` entries and both carry the exact AVB argument pair. Refuse to apply
this text substitution to a different structure:

```bash
SYSTEM_LINES=$(grep -Ec '^[[:space:]]*system[[:space:]]' "$FSTAB" || true)
AVB_SYSTEM_LINES=$(grep -Ec '^[[:space:]]*system[[:space:]].*avb=vbmeta,avb_keys=/avb' "$FSTAB" || true)

test "$SYSTEM_LINES" = 2 || {
  echo "ERROR: expected two system entries, found $SYSTEM_LINES"
  exit 1
}

test "$AVB_SYSTEM_LINES" = 2 || {
  echo "ERROR: expected the known AVB flags on both system entries, found $AVB_SYSTEM_LINES"
  exit 1
}

sed -i 's/,avb=vbmeta,avb_keys=\/avb//g' "$FSTAB"
```

Product removal is a separate decision and defaults to off. Change `REMOVE_PRODUCT` to `yes` only when `product_a`
was deliberately deleted from `super` and product content is supplied by the GSI system image:

```bash
REMOVE_PRODUCT=no  # Change to yes only after confirming the product-less layout.

case "$REMOVE_PRODUCT" in
  yes) sed -i '/^[[:space:]]*product[[:space:]]/d' "$FSTAB" ;;
  no)  echo 'Keeping the product fstab entry' ;;
  *)   echo "ERROR: REMOVE_PRODUCT must be yes or no"; exit 1 ;;
esac
```

Do **not** remove the product line when the logical product partition still exists or is required by that device.

Review the exact change:

```bash
diff -u "$WORK/fstab.before" "$FSTAB" || true
grep -nE '^[[:space:]]*(system|product)[[:space:]]' "$FSTAB"
```

Hard gate: refuse to continue if either `system` entry still carries AVB/dm-verity flags.

```bash
test "$(grep -Ec '^[[:space:]]*system[[:space:]]' "$FSTAB" || true)" = 2

if grep -E '^[[:space:]]*system[[:space:]].*(avb=|avb_keys=)' "$FSTAB"; then
  echo 'ERROR: system still enables AVB/dm-verity'
  exit 1
fi

if [ "$REMOVE_PRODUCT" = yes ] && grep -qE '^[[:space:]]*product[[:space:]]' "$FSTAB"; then
  echo 'ERROR: product entry was requested for removal but is still present'
  exit 1
fi
```

## 4. Repack the vendor ramdisk and vendor_boot

```bash
OUT="$PWD/vendor_boot.noverity.img"
test ! -e "$OUT"
test -s "$WORK/ramdisk.list"

MKARGS=$(python3 "$UNPACK" --boot_img "$IN" --format mkbootimg)
test -n "$MKARGS"

(
  cd "$ROOT_WRITE"
  cpio -o -H newc < "$WORK/ramdisk.list" > "$WORK/ramdisk.new.cpio"
)

test -s "$WORK/ramdisk.new.cpio"
lz4 -l -f "$WORK/ramdisk.new.cpio" "$WORK/out/vendor_ramdisk00"

(
  cd "$WORK"
  eval python3 "$MKBOOT" $MKARGS --vendor_boot "$OUT"
)

test -s "$OUT"
stat -c 'Rebuilt bytes: %s' "$OUT"
sha256sum "$OUT"
```

Do not flash yet.

## 5. Re-unpack and verify the rebuilt image

```bash
test ! -e "$WORK/verify"
mkdir -p "$WORK/verify/out" "$WORK/verify/root"
python3 "$UNPACK" --boot_img "$OUT" --out "$WORK/verify/out"
test -f "$WORK/verify/out/vendor_ramdisk00"

lz4 -d -f \
  "$WORK/verify/out/vendor_ramdisk00" \
  "$WORK/verify/ramdisk.cpio"

(
  cd "$WORK/verify/root"
  cpio -idm --no-absolute-filenames < "$WORK/verify/ramdisk.cpio"
)

VERIFY_FSTAB="$WORK/verify/root/first_stage_ramdisk/fstab.sun55iw3p1"
test -f "$VERIFY_FSTAB"
grep -nE '^[[:space:]]*(system|product)[[:space:]]' "$VERIFY_FSTAB"

test "$(grep -Ec '^[[:space:]]*system[[:space:]]' "$VERIFY_FSTAB" || true)" = 2

if grep -E '^[[:space:]]*system[[:space:]].*(avb=|avb_keys=)' "$VERIFY_FSTAB"; then
  echo 'ERROR: rebuilt image still enables dm-verity for system'
  exit 1
fi

if [ "$REMOVE_PRODUCT" = yes ] && grep -qE '^[[:space:]]*product[[:space:]]' "$VERIFY_FSTAB"; then
  echo 'ERROR: rebuilt image still contains the product entry'
  exit 1
fi
```

Confirm that the only intended semantic changes are:

- removal of `avb=vbmeta,avb_keys=/avb` from both `system` entries; and
- removal of the `product` entry only when `product_a` has also been deleted.

## 6. Pad the image to the partition size

Leave the root Bash shell started in Step 2, returning to the original host shell:

```bash
exit

SLOT=$(adb shell getprop ro.boot.slot_suffix | tr -d '\r')
case "$SLOT" in
  _a|_b) ;;
  *) echo "ERROR: unexpected slot suffix: '$SLOT'"; exit 1 ;;
esac

PART="/dev/block/by-name/vendor_boot${SLOT}"
adb shell "test -b '$PART'"

PART_BYTES=$(adb shell "blockdev --getsize64 '$PART'" | tr -d '\r')
case "$PART_BYTES" in
  ''|*[!0-9]*) echo "ERROR: invalid partition size: '$PART_BYTES'"; exit 1 ;;
esac
test "$PART_BYTES" -gt 0

test -s vendor_boot.noverity.img
test ! -e vendor_boot.noverity.padded.img
cp vendor_boot.noverity.img vendor_boot.noverity.padded.img

IMAGE_BYTES=$(stat -c %s vendor_boot.noverity.padded.img)
test "$IMAGE_BYTES" -le "$PART_BYTES"
truncate -s "$PART_BYTES" vendor_boot.noverity.padded.img
test "$(stat -c %s vendor_boot.noverity.padded.img)" = "$PART_BYTES"
stat -c 'Flash image bytes: %s' vendor_boot.noverity.padded.img
```

The padded image must exactly equal the reported partition size.

## 7. Write vendor_boot through rooted Android

```bash
REMOTE_FLASH=/data/local/tmp/vendor_boot.noverity.padded.img
adb shell "rm -f '$REMOTE_FLASH'"
adb push vendor_boot.noverity.padded.img "$REMOTE_FLASH"

adb shell "test \"\$(stat -c %s '$REMOTE_FLASH')\" = '$PART_BYTES'"

adb shell "dd \
  if='$REMOTE_FLASH' \
  of='$PART' \
  bs=4M"

adb shell sync
```

Do not reboot yet.

These pasteable commands require rooted adbd. If `adb shell id` is no longer uid 0, stop and restore root access before
continuing rather than improvising around the partition write.

## 8. Read the partition back and compare it before rebooting

```bash
REMOTE_READBACK=/data/local/tmp/vendor_boot.readback.img
test ! -e vendor_boot.readback.img
adb shell "rm -f '$REMOTE_READBACK'"

adb shell "dd \
  if='$PART' \
  of='$REMOTE_READBACK' \
  bs=4M"

adb pull "$REMOTE_READBACK" vendor_boot.readback.img

test "$(stat -c %s vendor_boot.noverity.padded.img)" = "$PART_BYTES"
test "$(stat -c %s vendor_boot.readback.img)" = "$PART_BYTES"

stat -c '%s %n' \
  vendor_boot.noverity.padded.img \
  vendor_boot.readback.img

sha256sum \
  vendor_boot.noverity.padded.img \
  vendor_boot.readback.img

cmp vendor_boot.noverity.padded.img vendor_boot.readback.img \
  && echo 'vendor_boot read-back verified'
```

Only reboot if `cmp` succeeds.

## 9. Reboot and confirm the result

```bash
adb reboot
adb wait-for-device
adb root
adb wait-for-device

SLOT=$(adb shell getprop ro.boot.slot_suffix | tr -d '\r')
case "$SLOT" in
  _a|_b) ;;
  *) echo "ERROR: unexpected slot suffix: '$SLOT'"; exit 1 ;;
esac

DM_TABLE=$(adb shell "dmctl table system${SLOT}")
printf '%s\n' "$DM_TABLE"

if grep -q 'verity' <<<"$DM_TABLE"; then
  echo 'ERROR: system still has a verity target'
  exit 1
fi

grep -q 'linear' <<<"$DM_TABLE"
echo 'system uses linear targets only'
```

Every target in the active `system` table should now be `linear`; none should say `verity`.

## Validation record

The complete decision, rebuild, and write/read-back command path was dry-run after publication:

- Part I was executed against the live, already-corrected C10. It read the active 32 MiB `vendor_boot`, unpacked it,
  found no AVB flags in either system entry, and correctly reached the **do not write** decision.
- Part II was executed offline against the preserved stock `vendor_boot` with SHA-256
  `95dc7e6cb526f3084fe83e8ce9cf5ca06136729513395279bf9f87564150f5dd`. The known AVB flags were detected,
  patched, rebuilt, re-unpacked, and verified.
- The padding, upload, `dd`, read-back, size checks, SHA-256 checks, and `cmp` sequence was executed against a 32 MiB
  mock file under `/data/local/tmp`; no real boot partition was written during the dry run.
- The final runtime check passed against the live C10 and reported only `linear` targets.

The actual device modification predates this dry run and was validated by successful GSI boots. The mock target was
used here specifically to test the published write/read-back commands without risking the working combined
vendor_boot.

## What this change does not do

- It does not modify vbmeta.
- It does not patch U-Boot, TOC0, or the signed early boot chain.
- It does not unlock fastbootd.
- It does not disable verification for vendor or other partitions.
- It does not use the C10 fastbootd binary offset.

The fixed `0x0a4d1c` offset documented elsewhere applies only to the separate fastbootd unlock patch. It is unrelated
to this fstab modification.

## Recovery

If the rebuilt image fails to boot, restore `vendor_boot.stock.img` through the previously validated FEL or recovery
path. Never begin Part II without that recovery path and an independently stored backup.

Do not flash a verification-disabled vbmeta as a substitute for this targeted fstab change, and do not re-lock a
device that depends on modified boot images.
