---
title: "Recreating a bcache Backing Device with a New Block Size (Cache Device Not Used)"
description: "This article describes how to change the bcache block size when you are operating without a cache..."
pubDatetime: 2026-01-19T11:27:01.086Z
---

This article describes how to change the **bcache block size** when you are operating **without a cache device** (the cache device has already been detached, and you will not add one during re-creation). Because bcache block size is defined at creation time, the practical approach is to **recreate the bcache backing device** with the desired parameters.

## Important Notes (Read First)

* Changing the bcache block size requires **recreating bcache metadata**. Treat this as a **destructive operation** unless you have verified recovery procedures and have a current backup.
* bcache stores data beginning at a **data offset** on the backing device. You must capture the current offset and reuse it during re-creation to keep layout alignment consistent.
* This procedure assumes:

  * The **cache device is already detached**.
  * You will create **only the backing device** (`-B`) and **will not** create or attach a cache device (`-C` / attach).

## Prerequisites

* Root privileges.
* `bcache-tools` installed (provides `bcache-super-show` and `make-bcache`).
* The backing device identified (example: `/dev/sdb1`).
* Any filesystems/LVM/dm-crypt/MD layers above `/dev/bcache*` will be stopped cleanly.



## Step 1 — Record the Current Data Offset

Read the existing bcache superblock on the backing device and record the data offset (commonly shown as `dev.data.first_sector` or an equivalent field):

```bash
sudo bcache-super-show /dev/sdb1
```

Make a note of the value that corresponds to the **data start offset**. You will reuse it as `<OFFSET>` in Step 4.

## Step 2 — Stop Using the bcache Device Cleanly

### 2.1 Unmount (and stop upper layers)

If `/dev/bcache0` (or similar) is mounted, unmount it:

```bash
mount | grep bcache
sudo umount /dev/bcache0
```

If you use LVM, dm-crypt, MD RAID, etc. on top of `/dev/bcache0`, stop those layers first.

### 2.2 Stop the bcache device

Stop the bcache device via sysfs (example for `/dev/bcache0`):

```bash
echo 1 | sudo tee /sys/block/bcache0/bcache/stop
```

If this fails, an upper layer is still holding the device open. Re-check mounts and dependent mappings.

## Step 3 — Remove Existing bcache Signatures from the Backing Device

Wipe the bcache signatures from the backing device:

```bash
sudo wipefs -a /dev/sdb1
```

This removes filesystem/RAID/bcache signatures that would otherwise conflict with re-creation.

## Step 4 — Recreate bcache on the Backing Device with a New Block Size

Now recreate the backing device bcache metadata, specifying:

* Your **new block size** (example: `4k`)
* The **previously recorded data offset** (`<OFFSET>`)
* Backing mode only (`-B`)

```bash
sudo make-bcache --block 4k --data-offset <OFFSET> -B /dev/sdb1
```

Notes:

* Replace `4k` with your target block size.
* Replace `<OFFSET>` with the value captured in Step 1.
* Because you are not using a cache device, **do not** run `make-bcache -C ...` and **do not** attach a cache set.

## Step 5 — Verify the New bcache Device

Confirm that `/dev/bcache*` has been created:

```bash
ls -l /dev/bcache*
lsblk -f | grep -E 'bcache|sdb1'
```

Check state (example for `/dev/bcache0`):

```bash
cat /sys/block/bcache0/bcache/state
```

At this point, you can create a filesystem (if appropriate) and mount it, or reintroduce your higher layers (LVM, dm-crypt, etc.) as required by your environment.



## Operational Consideration (Running Without a Cache Device)

If you do not add a cache device, bcache effectively behaves as a pass-through block device wrapper. This can still be useful if you plan to attach cache later or want consistent device naming/management, but you should be aware that **you are not gaining caching benefits** until a cache set is attached.



## Common Failure Modes and How to Avoid Them

* **“Device or resource busy” when stopping bcache**
  An upper layer (mount, LVM, dm-crypt, MD, etc.) is still open. Stop/unmount from the top down.
* **Wrong offset used during re-creation**
  Using an incorrect `<OFFSET>` can cause data layout mismatch and potential data loss. Always record the offset directly from `bcache-super-show`.



## Summary

To change bcache block size without using a cache device, you:

1. Record the **data offset** from `bcache-super-show`.
2. Cleanly stop usage and **stop** the bcache device.
3. Remove signatures with `wipefs -a`.
4. Recreate bcache **backing only** using `make-bcache --block <new> --data-offset <old> -B`.
5. Verify `/dev/bcache*` and proceed with your storage stack.
