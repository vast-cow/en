---
title: "How to Check and Configure bcache `discard` and `cache_mode`"
description: "bcache lets you use a fast device, such as an SSD, as a cache for a slower storage device, such as an..."
pubDatetime: 2026-05-03T07:20:14.503Z
---

bcache lets you use a fast device, such as an SSD, as a cache for a slower storage device, such as an HDD. This can help improve storage performance while still using larger, lower-cost disks for the main data storage.

This article explains how to check and configure two common bcache settings: `discard` and `cache_mode`.

## Checking and Enabling `discard`

The `discard` setting controls whether bcache can notify the storage device about blocks that are no longer in use. When an SSD is involved, this is commonly related to TRIM behavior.

You can check the current `discard` setting with the following command:

```bash
cat /sys/block/sda/bcache/discard
```

To enable `discard`, write `1` to the setting:

```bash
echo 1 | sudo tee /sys/block/sda/bcache/discard
```

Once enabled, bcache can pass discard information to the underlying storage device when appropriate.

## Checking and Changing `cache_mode`

The `cache_mode` setting controls how bcache handles write operations.

To check the current cache mode, run:

```bash
cat /sys/block/bcache0/bcache/cache_mode
```

To change the cache mode to `writethrough`, use this command:

```bash
echo writethrough | sudo tee /sys/block/bcache0/bcache/cache_mode
```

In `writethrough` mode, data is written both to the cache and to the backing storage device. This makes it a practical choice when you want a safer write behavior that does not rely only on the cache device.

## Things to Check Before Running These Commands

These commands write directly to bcache settings under `/sys`. Before running them, make sure the device names match your actual system.

For example, your system may not use `sda` or `bcache0`. The correct device names can vary depending on your storage layout and how bcache was configured.

## Summary

The `discard` setting allows bcache to notify storage devices about unused blocks, which can be useful in SSD-based environments.

The `cache_mode` setting controls bcache’s write behavior. `writethrough` is a straightforward and relatively safe mode because writes are sent to both the cache and the backing device.

In normal use, it is best to check the current settings first, confirm the target devices, and then apply any changes carefully.
