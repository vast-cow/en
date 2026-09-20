---
title: "Setting up RHEL to boot with `/` (root) on an `mdadm` RAID device"
description: "When you place the RHEL root filesystem on a software RAID array built with mdadm, the initramfs..."
pubDatetime: 2026-01-21T09:28:26.048Z
---

When you place the RHEL root filesystem on a software RAID array built with `mdadm`, the initramfs (dracut) must be able to find and assemble that RAID array very early in the boot process. A reliable way to do this is to pass the RAID array UUID to the kernel command line via GRUB, so dracut knows exactly which array to assemble.

Below is a minimal and practical configuration workflow.



## 1) Confirm the RAID array UUID

First, obtain the array definition and its UUID from `mdadm`.

```bash
$ sudo mdadm --detail --scan
ARRAY /dev/md/0 metadata=1.2 UUID=XXX
```

Key point:

* The `UUID=XXX` here is the identifier you will pass to the boot process.
* Using the UUID is preferred over guessing device names, which can change depending on discovery order.



## 2) Add the RAID UUID to the GRUB kernel command line

Edit GRUB’s default configuration and append `rd.md.uuid=XXX` to the kernel parameters.

```bash
$ sudo -e /etc/default/grub
GRUB_CMDLINE_LINUX=".. rd.md.uuid=XXX"
```

What this does:

* `rd.md.uuid=XXX` is a dracut parameter. It tells the initramfs: “assemble the md RAID array that has this UUID during early boot.”
* This is critical when root is on RAID, because the system cannot mount `/` until the array is assembled.

Notes:

* Keep the existing kernel parameters (`..` in your example) and simply add the new one.
* If you have multiple md arrays required for boot, you can specify multiple `rd.md.uuid=` entries.



## 3) Regenerate GRUB configuration and update BLS entries

On modern RHEL, Boot Loader Specification (BLS) entries are commonly used. The following command regenerates GRUB config and updates the BLS kernel command line:

```bash
$ sudo grub2-mkconfig -o /etc/grub2.cfg --update-bls-cmdline
```

Why this matters:

* Without updating the BLS command line, your new `rd.md.uuid=...` parameter might not actually be applied at boot, depending on how the system is configured.
* This step ensures the runtime boot entries reflect your change.



## Outcome and quick validation

After these steps, the bootloader will pass `rd.md.uuid=XXX` to dracut, dracut will assemble `/dev/md/0` early, and the system will be able to mount the root filesystem from that md RAID device.

If you want a quick sanity check after reboot, you can confirm the array is active:

```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md/0
```
