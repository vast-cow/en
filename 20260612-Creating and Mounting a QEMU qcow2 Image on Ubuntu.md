---
title: "Creating and Mounting a QEMU qcow2 Image on Ubuntu"
description: "Prerequisites   In this guide, a “QEMU COW image” is treated as the commonly used qcow2..."
pubDatetime: 2026-06-12T03:17:47.191Z
---

## Prerequisites

In this guide, a “QEMU COW image” is treated as the commonly used **qcow2** format. QEMU primarily supports the `raw` and `qcow2` disk image formats, and `qcow2` provides features such as sparse allocation, compression, and multiple snapshots. ([QEMU][1])

On Ubuntu, the standard approach is to use `qemu-img` and `qemu-nbd`. Ubuntu man pages indicate that both `qemu-img` and `qemu-nbd` are included in the `qemu-utils` package. ([Ubuntu Manpages][2])

---

## 0. Important Notes

When mounting a disk image from an existing VM, the VM should generally be **powered off**. The official QEMU documentation warns that modifying an image with `qemu-img` while it is being used by a running VM or another process can corrupt the image. ([GitLab][3])

Also, avoid mounting untrusted qcow2 images directly on the host OS. The `qemu-nbd` documentation warns that a malicious guest image could potentially exploit kernel bugs during partition detection or filesystem mounting. ([QEMU][4])

---

## 1. Install Required Packages

```bash
sudo apt update
sudo apt install -y qemu-utils parted e2fsprogs
```

Main tools:

| Command     | Purpose                                                    |
| ----------- | ---------------------------------------------------------- |
| `qemu-img`  | Create, inspect, convert, and resize qcow2 images          |
| `qemu-nbd`  | Expose a qcow2 image as a block device such as `/dev/nbd0` |
| `parted`    | Create partitions                                          |
| `mkfs.ext4` | Create an ext4 filesystem                                  |

`qemu-nbd` exports a QEMU disk image through NBD and attaches it as a Linux block device such as `/dev/nbdX`. ([Ubuntu Manpages][5])

---

# Method A: Create and Mount a Partitioned qcow2 Image

If you want a layout similar to a normal virtual disk, this is the recommended approach.

## 2. Create a qcow2 Image

For example, create a 20 GB qcow2 disk:

```bash
mkdir -p ~/qemu-images
cd ~/qemu-images

qemu-img create -f qcow2 disk.qcow2 20G
qemu-img info disk.qcow2
```

According to the QEMU documentation, `qemu-img create` creates disk images and supports size suffixes such as `M` and `G`. ([QEMU][1])

---

## 3. Load the NBD Kernel Module

```bash
sudo modprobe nbd max_part=16
```

`max_part=16` allows partition devices such as `/dev/nbd0p1` and `/dev/nbd0p2` to be created.

---

## 4. Connect the qcow2 Image to `/dev/nbd0`

```bash
sudo qemu-nbd --connect=/dev/nbd0 --format=qcow2 disk.qcow2
```

Equivalent short form:

```bash
sudo qemu-nbd -c /dev/nbd0 -f qcow2 disk.qcow2
```

The official QEMU examples show exporting a qcow2 file as `/dev/nbd0`, with partitions appearing as devices such as `/dev/nbd0p1`. ([QEMU][4])

Verify:

```bash
lsblk /dev/nbd0
```

---

## 5. Create a Partition

Create a GPT partition table and a single ext4 partition occupying the entire disk:

```bash
sudo parted -s /dev/nbd0 mklabel gpt
sudo parted -s -a optimal /dev/nbd0 mkpart primary ext4 1MiB 100%
sudo partprobe /dev/nbd0
```

Verify:

```bash
lsblk /dev/nbd0
```

Expected output:

```text
nbd0      43:0    0   20G  0 disk
└─nbd0p1  43:1    0   20G  0 part
```

If `/dev/nbd0p1` does not appear, disconnect and reconnect:

```bash
sudo qemu-nbd --disconnect /dev/nbd0
sudo qemu-nbd --connect=/dev/nbd0 --format=qcow2 disk.qcow2
lsblk /dev/nbd0
```

---

## 6. Create a Filesystem

```bash
sudo mkfs.ext4 -L qemu-data /dev/nbd0p1
```

Running `mkfs` destroys any existing data. Use it only when creating a new filesystem.

---

## 7. Mount the Filesystem

```bash
sudo mkdir -p /mnt/qcow2
sudo mount /dev/nbd0p1 /mnt/qcow2
```

Verify:

```bash
df -h /mnt/qcow2
mount | grep /mnt/qcow2
```

Write a test file:

```bash
echo "hello qcow2" | sudo tee /mnt/qcow2/hello.txt
ls -l /mnt/qcow2
```

---

## 8. Unmount and Disconnect

Always unmount before disconnecting the NBD device:

```bash
sync
sudo umount /mnt/qcow2
sudo qemu-nbd --disconnect /dev/nbd0
```

Short form:

```bash
sudo qemu-nbd -d /dev/nbd0
```

Optionally unload the NBD module:

```bash
sudo rmmod nbd
```

---

# Method B: Use the Entire qcow2 Image Without Partitions

If the image is only being used as a simple data container, you can create a filesystem directly on the qcow2 image without partitioning. However, Method A is generally more natural for VM disks.

```bash
cd ~/qemu-images

qemu-img create -f qcow2 fs.qcow2 10G

sudo modprobe nbd max_part=8
sudo qemu-nbd -c /dev/nbd0 -f qcow2 fs.qcow2

sudo mkfs.ext4 -L qemu-fs /dev/nbd0

sudo mkdir -p /mnt/qcow2
sudo mount /dev/nbd0 /mnt/qcow2
```

Cleanup:

```bash
sync
sudo umount /mnt/qcow2
sudo qemu-nbd -d /dev/nbd0
```

---

# Mounting an Existing qcow2 Image

When inspecting an existing VM disk or cloud image, it is safest to attach it read-only first:

```bash
sudo modprobe nbd max_part=16
sudo qemu-nbd --read-only --connect=/dev/nbd0 --format=qcow2 existing.qcow2
```

Short form:

```bash
sudo qemu-nbd -r -c /dev/nbd0 -f qcow2 existing.qcow2
```

Inspect partitions:

```bash
lsblk -f /dev/nbd0
sudo fdisk -l /dev/nbd0
```

If `/dev/nbd0p1` contains an ext4 filesystem, mount it read-only:

```bash
sudo mkdir -p /mnt/qcow2
sudo mount -o ro /dev/nbd0p1 /mnt/qcow2
```

Cleanup:

```bash
sudo umount /mnt/qcow2
sudo qemu-nbd -d /dev/nbd0
```

To mount read-write, remove `--read-only` or `-r`. Ensure the VM is powered off and no other process is using the image.

---

# Creating a COW Overlay Image

If “COW image” refers to a **differencing image** that stores only changes relative to a base image, create a qcow2 image with a backing file.

Example:

```bash
qemu-img create -f qcow2 -F qcow2 -b base.qcow2 overlay.qcow2
```

Inspect:

```bash
qemu-img info --backing-chain overlay.qcow2
```

In this configuration, `overlay.qcow2` stores only differences from `base.qcow2`. The backing file itself is normally not modified. The `qemu-img create` documentation explains that specifying a backing file causes the new image to record only changes relative to that backing image. ([GitLab][3])

To mount the overlay image, connect the overlay—not the base image:

```bash
sudo modprobe nbd max_part=16
sudo qemu-nbd -c /dev/nbd0 -f qcow2 overlay.qcow2

lsblk -f /dev/nbd0
sudo mkdir -p /mnt/qcow2
sudo mount /dev/nbd0p1 /mnt/qcow2
```

Cleanup:

```bash
sync
sudo umount /mnt/qcow2
sudo qemu-nbd -d /dev/nbd0
```

If the backing image is raw:

```bash
qemu-img create -f qcow2 -F raw -b base.raw overlay.qcow2
```

---

## Minimal Example

Create a new qcow2 image, create an ext4 partition, and mount it:

```bash
sudo apt update
sudo apt install -y qemu-utils parted e2fsprogs

qemu-img create -f qcow2 disk.qcow2 20G

sudo modprobe nbd max_part=16
sudo qemu-nbd -c /dev/nbd0 -f qcow2 disk.qcow2

sudo parted -s /dev/nbd0 mklabel gpt
sudo parted -s -a optimal /dev/nbd0 mkpart primary ext4 1MiB 100%
sudo partprobe /dev/nbd0

sudo mkfs.ext4 /dev/nbd0p1

sudo mkdir -p /mnt/qcow2
sudo mount /dev/nbd0p1 /mnt/qcow2

# Work with the filesystem
echo test | sudo tee /mnt/qcow2/test.txt

# Cleanup
sync
sudo umount /mnt/qcow2
sudo qemu-nbd -d /dev/nbd0
```

[1]: https://www.qemu.org/docs/master/system/images.html "Disk Images — QEMU documentation"
[2]: https://manpages.ubuntu.com/manpages/noble/man1/qemu-img.1.html "Ubuntu Manpage: qemu-img - QEMU disk image utility"
[3]: https://qemu-project.gitlab.io/qemu/tools/qemu-img.html "QEMU disk image utility — QEMU documentation"
[4]: https://www.qemu.org/docs/master/tools/qemu-nbd.html "QEMU Disk Network Block Device Server — QEMU documentation"
[5]: https://manpages.ubuntu.com/manpages/focal/man8/qemu-nbd.8.html "Ubuntu Manpage: qemu-nbd - QEMU Disk Network Block Device Server"
