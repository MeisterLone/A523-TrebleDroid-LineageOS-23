# Checking and disabling dm-verity on the FancyDay C10

Before flashing a GSI, you need to answer one question: **does the first-stage `system` entry still turn on
dm-verity?**

On the stock FancyDay C10 firmware, it does. A GSI has a different hashtree from the stock system, so leaving that
check enabled produces an early bootloop. The good news is that checking is safe, and most of this guide is just
looking before touching anything.

## Quick answer

| What you find | What to do |
|---|---|
| `dmctl` shows only `linear`, and the first-stage system lines have no `avb=`/`avb_keys=` | You are already good. Stop after Part 1. |
| Either first-stage system line contains the known AVB flags | Use Part 2 before flashing the GSI. |
| The ramdisk, fstab path, or AVB layout looks different | Stop. Your firmware needs its own analysis. |

The guide is split on purpose:

1. **Part 1 is read-only.** It checks the live mapping and inspects a copy of `vendor_boot` on your computer.
2. **Part 2 writes `vendor_boot`.** Go there only when Part 1 proves the change is needed.

> **Device-specific note:** The paths below are verified on the FancyDay C10 (`sun55iw3p1`). Other A523 tablets may
> arrange their ramdisk differently. If an expected file is missing, that is a stop sign—not an invitation to flash a
> disabled vbmeta and hope for the best.

## What you need

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

## Part 1: Check first (safe and read-only)

Everything in Part 1 reads device state or copies data to the host. It does **not** write to `vendor_boot`, vbmeta,
or any other boot partition.

### 1. Make sure ADB is talking to the right tablet

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

### 2. See how `system` is mapped right now

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

This is a useful first look. The fstab check below is the final word, because that is what early boot reads next time.

### 3. Copy `vendor_boot` to your computer

This copies the active partition straight to your Linux computer. Nothing is written back to the tablet.

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

### 4. Unpack the copy and find the real first-stage fstab

These variables simply point at the two AOSP tools cloned earlier. The commands expect `mkbootimg/` in your current
directory.

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

`WORK` is just the temporary folder holding everything from here on. Keep this terminal open so the variable stays
set. If you close it, rerun this step instead of guessing the generated folder name.

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

An absent mapper is only a clue. Remove the product line later only if you know `product_a` was deliberately deleted
and the GSI supplies that content itself.

### 5. Decision time

Use the fstab output—not the device model name—as the final decision gate.

#### You are done—do not change `vendor_boot`—when

- neither `system` line contains `avb=` or `avb_keys=`; and
- the runtime `system` device-mapper table contains only `linear` targets.

In that state, this particular dm-verity fix is unnecessary.

#### You need Part 2 when

- either first-stage `system` line contains `avb=vbmeta`, `avb_keys=/avb`, or equivalent dm-verity configuration.

The GSI's ext4 data and hashtree cannot match the stock `vbmeta_system` root digest. On this firmware, leaving those
flags enabled causes a guaranteed early bootloop.

#### Stop and investigate when

- `vendor_boot` does not unpack normally;
- `vendor_ramdisk00` is absent;
- `first_stage_ramdisk/fstab.sun55iw3p1` is absent;
- the fstab uses a different AVB arrangement than the two known C10 `system` entries; or
- there is no tested recovery route.

Different layout, different job. Do not force the C10 paths onto it.

---

<details>
<summary><strong>Part 2: Fix it only if Part 1 says you need to</strong></summary>

> **This is the point where writing begins.** Continue only if Part 1 found the AVB flags, your backup is good, and
> you already know how you would restore `vendor_boot` if the tablet does not boot.

Keep using the same terminal from Part 1. The archive is extracted again as root so Android's original numeric owners
and file modes survive the rebuild.

### 1. Make one more backup

```bash
test -f vendor_boot.stock.img
test ! -e vendor_boot.stock.backup.img
cp -p vendor_boot.stock.img vendor_boot.stock.backup.img
sha256sum vendor_boot.stock.img vendor_boot.stock.backup.img
cmp vendor_boot.stock.img vendor_boot.stock.backup.img
```

Keep that backup somewhere off the tablet. It is your way back.

### 2. Open a clean root-owned copy of the ramdisk

Make sure the files from Part 1 are still there, then open a root shell with those paths carried across:

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

You are now in a root Bash shell. Make a fresh extraction next to the read-only one:

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

If `root-write` already exists, the command stops instead of mixing old and new files.

### 3. Remove the two dm-verity flags

The C10 has exactly two `system` lines, and both carry the same AVB pair. These checks stop the edit if your file does
not match that layout:

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

Product is a separate choice, so the default is safely `no`. Set it to `yes` only if you deliberately deleted
`product_a` and the GSI carries product content inside system:

```bash
REMOVE_PRODUCT=no  # Change to yes only after confirming the product-less layout.

case "$REMOVE_PRODUCT" in
  yes) sed -i '/^[[:space:]]*product[[:space:]]/d' "$FSTAB" ;;
  no)  echo 'Keeping the product fstab entry' ;;
  *)   echo "ERROR: REMOVE_PRODUCT must be yes or no"; exit 1 ;;
esac
```

If you are unsure, leave product alone.

Now look at the exact edit:

```bash
diff -u "$WORK/fstab.before" "$FSTAB" || true
grep -nE '^[[:space:]]*(system|product)[[:space:]]' "$FSTAB"
```

One last gate before rebuilding:

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

### 4. Repack `vendor_boot`

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

Still do not flash it. First prove the rebuilt image contains the edit.

### 5. Unpack your rebuilt image and check it again

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

At this point the only meaningful changes should be the two AVB removals, plus the product-line removal only when you
explicitly requested it.

### 6. Pad the image to the real partition size

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

The padded file should now be exactly the same size as the partition.

### 7. Write it, but do not reboot yet

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

Do not reboot yet. Read it back first.

These commands expect rooted adbd. If `adb shell id` is no longer uid 0, stop and fix that before continuing.

### 8. Read it back and prove the write was exact

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

If `cmp` says the files differ, do not reboot. Restore the backup while Android is still running.

### 9. Reboot and do one last check

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

That is the result you want: `linear` targets and no `verity` target.

<details>
<summary><strong>How these commands were tested</strong></summary>

The complete decision, rebuild, and write/read-back path was dry-run after publication:

- Part 1 ran against the live, already-corrected C10 and correctly stopped at **do not write**.
- Part 2 ran offline against the preserved stock `vendor_boot` with SHA-256
  `95dc7e6cb526f3084fe83e8ce9cf5ca06136729513395279bf9f87564150f5dd`.
- Repacking, re-unpacking, padding, upload, `dd`, read-back, SHA-256, and `cmp` all passed. The write test used a
  32 MiB mock file under `/data/local/tmp`, not the real boot partition.
- The live C10's final table reports only `linear` targets.

The real device change predates this dry run and has also been validated by successful GSI boots.

</details>

## What this does—and does not—change

- It removes dm-verity from the two first-stage `system` entries.
- It optionally removes the product entry when you explicitly choose the product-less layout.
- It does not modify vbmeta.
- It does not patch U-Boot, TOC0, or the signed early boot chain.
- It does not unlock fastbootd.
- It does not disable verification for vendor or other partitions.
- It does not use the C10 fastbootd binary offset.

The fixed `0x0a4d1c` offset documented elsewhere applies only to the separate fastbootd unlock patch. It is unrelated
to this fstab modification.

## If it does not boot

If the rebuilt image does not boot, restore `vendor_boot.stock.img` through the FEL or recovery path you tested
before starting. That is exactly why the backup comes first.

Do not substitute a verification-disabled vbmeta for this small fstab change, and do not re-lock a device that depends
on modified boot images.

</details>
