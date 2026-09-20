---
title: "How to Force bcache to Recognize a Backing Device Without a Cache Device"
description: "In an environment using bcache, if the cache device fails, disappears, or is disconnected, the bcache..."
pubDatetime: 2026-06-18T09:14:43.407Z
---

In an environment using bcache, if the cache device fails, disappears, or is disconnected, the bcache device (such as `/dev/bcache0`) may not be created automatically from the backing device alone.

In this situation, you can register the backing device with bcache and then write `1` to `running` to force the backing device to start even without a cache device.

## Prerequisites

This article assumes the following backing device:

```bash
/dev/sdd4
```

Replace `/dev/sdd4` with the appropriate device for your environment.

## Procedure

### 1. Load the bcache kernel module

```bash
sudo modprobe bcache
```

### 2. Register the backing device with bcache

```bash
echo /dev/sdd4 | sudo tee /sys/fs/bcache/register
```

This makes bcache recognize the backing device.

### 3. Force startup without a cache device

```bash
echo 1 | sudo tee /sys/class/block/sdd/sdd4/bcache/running
```

This creates a bcache device such as `/dev/bcache0`.

Verify it:

```bash
lsblk
ls -l /dev/bcache*
```

The created device will typically be named `/dev/bcache0`, `/dev/bcache1`, and so on.

## Which Device to Mount

Mount the generated bcache device, not the backing device itself.

For example, if `/dev/bcache0` is created:

```bash
sudo mount /dev/bcache0 /mnt
```

Do not mount `/dev/sdd4` directly; always use the bcache device such as `/dev/bcache0`.

## About the sysfs Path

When a partition is used as the backing device, `running` is located at:

```bash
/sys/class/block/sdd/sdd4/bcache/running
```

In this example, the disk is `/dev/sdd` and the partition is `/dev/sdd4`, so this is the correct path.

For NVMe devices, if the backing device is `/dev/nvme0n1p3`, the path would be:

```bash
/sys/class/block/nvme0n1/nvme0n1p3/bcache/running
```

## Summary

The minimum steps required to force bcache to recognize a backing device when no cache device is available are:

```bash
sudo modprobe bcache
echo /dev/sdd4 | sudo tee /sys/fs/bcache/register
echo 1 | sudo tee /sys/class/block/sdd/sdd4/bcache/running
```

Afterward, mount the generated bcache device:

```bash
sudo mount /dev/bcache0 /mnt
```

## Important Notes

If writeback caching was being used, dirty data may still exist on the cache device.

Forcing the backing device to start without the cache device in that situation can result in filesystem inconsistencies or data corruption.

Be especially careful if:

* The cache mode was `writeback`
* The cache device failed unexpectedly
* Dirty data was not flushed before shutdown
* The consistency between the backing device and cache device is unknown

If you want to inspect the data safely, mount it read-only:

```bash
sudo mount -o ro /dev/bcache0 /mnt
```

For data recovery purposes, it is safest to mount the filesystem read-only first and copy important data elsewhere before attempting any repairs or modifications.
