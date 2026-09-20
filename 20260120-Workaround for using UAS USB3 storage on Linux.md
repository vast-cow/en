---
title: "Workaround for using UAS USB3 storage on Linux"
description: "Identify problematic USB storage bridges and apply temporary or persistent quirks to disable UAS or limit transfer sizes, including multiple-device settings and initramfs updates."
pubDatetime: 2026-01-20T01:53:13.029Z
updatedDate: 2026-01-20T02:11:12.527Z
---

Some USB-to-SATA/NVMe bridge chips advertise UAS (USB Attached SCSI) but behave poorly under the `uas` driver (random I/O errors, resets, disconnects, poor SMART pass-through, etc.). A common workaround is to apply a **usb-storage quirk** so the device uses the classic `usb-storage` driver instead of `uas`.

This guide covers:

* How to find the **Vendor ID (VID)** and **Product ID (PID)**
* How to apply quirks **temporarily** via sysfs
* How to apply quirks **persistently** via `modprobe.d`
* How to specify **multiple devices**
* What the quirk flags **`u`** and **`m`** mean
* When you may need to rebuild **initramfs** (`dracut` / `update-initramfs`)

## Identify Storage

```sh
lsusb -tv
# view USB tree and the kernel driver in use (uas vs usb-storage)
# view VID:PID
```

Tip: `lsusb -tv` is the easiest way to confirm which driver the kernel actually bound to the device.

## Quirk Flags (What `u` and `m` mean)

The `usb-storage` module accepts a `quirks=` parameter containing one or more entries in the form:

```
VID:PID:FLAGS
```

Common flags include:

* **`u` = IGNORE_UAS**
  Forces the kernel to **avoid binding the `uas` driver** for this device and use **`usb-storage` (BOT)** instead.

* **`m` = MAX_SECTORS_64**
  Limits transfer size to **64 sectors (32KB)**. This can improve stability for devices/bridges that misbehave with larger transfers (at the cost of performance).

A frequently used combination is **`mu`**: disable UAS *and* cap transfer size for stability.

## Temporary Settings (runtime / non-persistent)

Writing to sysfs updates the **currently running kernel** only. It will be lost on reboot.

### Single device

```sh
echo "0bc2:231a:u" | sudo tee /sys/module/usb_storage/parameters/quirks

# disconnect and reconnect
```

### Multiple devices (comma-separated)

Use a single line with comma-separated entries:

```sh
printf '0bc2:231a:u,6795:2756:mu\n' | sudo tee /sys/module/usb_storage/parameters/quirks

# disconnect and reconnect
```

Notes:

* Use **commas**, no spaces.
* This **overwrites** the current value (it does not append).

### Add a device without losing existing entries

Because sysfs does not support append mode, read the current value, add your entry, and write it back:

```sh
cur=$(cat /sys/module/usb_storage/parameters/quirks)
new='0bc2:231a:u'

if [ -z "$cur" ]; then
  printf '%s\n' "$new" | sudo tee /sys/module/usb_storage/parameters/quirks >/dev/null
else
  printf '%s,%s\n' "$cur" "$new" | sudo tee /sys/module/usb_storage/parameters/quirks >/dev/null
fi

# disconnect and reconnect
```

## Persistent Settings (recommended)

To make the quirk survive reboots, set it via `modprobe.d`.

Create e.g. `/etc/modprobe.d/usb-storage-quirks.conf`:

```conf
options usb-storage quirks=0bc2:231a:u,6795:2756:mu
```

Then reboot, or reload modules (reboot is typically simplest and most reliable for USB storage bindings).

## initramfs note (dracut / update-initramfs)

In many cases, adding the `modprobe.d` file and rebooting is sufficient.

However, if your system loads storage drivers early from **initramfs** (for example, booting from USB storage, or using LUKS/LVM on the affected device), the new `modprobe.d` settings may not be applied during early boot unless you **rebuild initramfs**.

### RHEL/CentOS/Rocky/Alma/Fedora (dracut)

```sh
sudo dracut -f
sudo reboot
```

### Debian/Ubuntu (initramfs-tools)

```sh
sudo update-initramfs -u
sudo reboot
```

Notes:

* `grub2-mkconfig` is typically **not** required for `modprobe.d` changes (it is for GRUB menu/kernel command-line changes, not module option files).
* Even after rebuilding initramfs, you may still need to disconnect/reconnect the device (or reboot) so the driver binding is re-evaluated.

## Confirm

Confirm the device is using **usb-storage** (not `uas`):

```sh
lsusb -tv
```

Look for a line indicating something like `Driver=usb-storage` for your device path.

## ref

* [https://www.smartmontools.org/wiki/SAT-with-UAS-Linux](https://www.smartmontools.org/wiki/SAT-with-UAS-Linux)
