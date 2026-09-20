---
title: "Limiting SATA Link Speed to 3 Gbps on Linux Using libata.force"
description: "You can use the Linux kernel boot parameter libata.force to limit the link speed of a specific ATA..."
pubDatetime: 2026-07-30T13:55:58.304Z
---

You can use the Linux kernel boot parameter `libata.force` to **limit the link speed of a specific ATA port to SATA 3 Gbps (Gen2) instead of 6 Gbps (Gen3)**.

## Example Configuration

If the target port is recognized as `ata2` in Linux, add the following kernel parameter:

```text
libata.force=2:3.0G
```

The format is `PORT:SPEED`, where `PORT` is the numeric portion of `ata2`. To configure multiple ports, separate them with commas:

```text
libata.force=2:3.0G,5:3.0G
```

According to the official kernel documentation, `libata.force=[ID:]VAL` can be used to limit the SATA link speed on a per-port basis to either `1.5Gbps` or `3.0Gbps`. ([Linux Kernel Documentation][1])

## Identifying the Target Port

First, check the boot log to determine which ATA port the disk is connected to:

```bash
sudo dmesg | grep -E 'ata[0-9]+: SATA link'
```

Example output:

```text
ata2: SATA link up 6.0 Gbps
ata5: SATA link up 6.0 Gbps
```

You can also verify the mapping to block devices with:

```bash
ls -l /dev/disk/by-path/ | grep ata
```

The important point is to specify the **`ataN` number reported by Linux**, not `/dev/sda` or the motherboard's physical "SATA Port 2" label.

## Making the Setting Persistent with GRUB

Add the parameter to the existing kernel command line in `/etc/default/grub`:

```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash libata.force=2:3.0G"
```

On Debian- and Ubuntu-based systems, then run:

```bash
sudo update-grub
sudo reboot
```

On distributions that generate the GRUB configuration differently, use either `grub-mkconfig` or `grub2-mkconfig`, depending on your environment.

## Verifying the Configuration

After rebooting:

```bash
cat /proc/cmdline
```

Then verify the negotiated link speed:

```bash
sudo dmesg | grep 'ata2: SATA link'
```

Expected output:

```text
ata2: SATA link up 3.0 Gbps
```

You can also check via sysfs:

```bash
cat /sys/class/ata_link/link2/sata_spd
cat /sys/class/ata_link/link2/sata_spd_limit
```

`/sys/class/ata_link/.../sata_spd` reports the current link speed, while `sata_spd_limit` reports the maximum speed imposed by `libata`. These files are read-only, so the limit is normally configured through the kernel boot parameter rather than changed at runtime with `echo`. ([Linux Kernel Archive][2])

## Notes

* This does **not** throttle bandwidth; it limits the **maximum SATA PHY link negotiation speed** to 3 Gbps.
* The `ataN` numbering may change if controllers are added, BIOS settings are modified, or the kernel or drivers change.
* If your BIOS/UEFI provides a per-port "SATA Speed" or "Gen2" option, that may be a clearer way to configure the physical port.
* This method may not apply to disks connected through USB-to-SATA adapters, NVMe, SAS, or hardware RAID controllers.

[1]: https://docs.kernel.org/admin-guide/kernel-parameters.html "The kernel’s command-line parameters — The Linux Kernel documentation"
[2]: https://www.kernel.org/doc/Documentation/ABI/testing/sysfs-ata?utm_source=chatgpt.com "sysfs-ata"
