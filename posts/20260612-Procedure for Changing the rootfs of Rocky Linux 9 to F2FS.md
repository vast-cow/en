---
title: "Procedure for Changing the rootfs of Rocky Linux 9 to F2FS"
description: "Migrate a Rocky Linux root filesystem to F2FS using an ELRepo kernel, metadata-preserving backup, initramfs and boot configuration updates, and SELinux relabeling."
pubDatetime: 2026-06-12T04:24:49.042Z
---

# Procedure for Changing the rootfs of Rocky Linux 9 to F2FS

When changing the root filesystem of Rocky Linux 9 to F2FS, it is difficult to complete the process using only the standard installer. In practice, the realistic workflow is to first perform a normal Rocky 9 installation using ext4, XFS, or similar, then boot from a Live USB, back up the rootfs, recreate it as F2FS, and restore it.

This article organizes the procedure under the following assumptions:

* `/boot` is a separate partition from the rootfs
* `/boot` remains ext4 or XFS
* Only `/` is changed to F2FS
* ELRepo’s `kernel-lt` is used instead of the standard Rocky 9 kernel
* The same UUID as the original rootfs is specified when running `mkfs.f2fs`
* If the system does not boot, recovery is performed by chrooting from an Ubuntu Live USB

ELRepo is a repository that provides kernels, filesystem drivers, and similar packages for Enterprise Linux, and `kernel-lt` is provided as a long-term support kernel. When trying F2FS root on Rocky/RHEL 9 systems, assuming the use of an ELRepo kernel rather than the standard kernel is practical. ([Elrepo][1])

---

## 1. Basic Policy

The final configuration will be as follows.

```text
/boot      ext4 or XFS
/boot/efi  vfat
/          f2fs
```

The reason for not making `/boot` F2FS is simple: to reduce uncertainty around GRUB and early boot. If only the rootfs is F2FS, GRUB reads the kernel and initramfs from `/boot` as usual, and the rootfs is mounted by the F2FS driver inside the initramfs.

---

## 2. Install Rocky 9 Normally

First, install Rocky Linux 9 normally.

At this point, the rootfs may be ext4, XFS, or similar.

Example partition layout:

```text
/dev/nvme0n1p1  /boot/efi
/dev/nvme0n1p2  /boot
/dev/nvme0n1p3  /
```

What matters is **keeping `/boot` separate from `/`**.

---

## 3. Set SELinux to permissive

After conversion to F2FS, there is a possibility of running into issues involving SELinux labels or extended attributes. Therefore, set SELinux to permissive before the work.

```bash
sudo vi /etc/selinux/config
```

```ini
SELINUX=permissive
```

Reboot after making the change.

```bash
sudo reboot
```

After booting, check the status.

```bash
getenforce
```

```text
Permissive
```

---

## 4. Install ELRepo’s kernel-lt

Use ELRepo’s `kernel-lt` instead of the standard Rocky 9 kernel.

```bash
sudo dnf install https://www.elrepo.org/elrepo-release-9.el9.elrepo.noarch.rpm
sudo dnf --enablerepo=elrepo-kernel install kernel-lt
```

Caution is required if Secure Boot is enabled. According to ELRepo’s explanation, `kernel-ml` and `kernel-lt` are not signed with a Secure Boot key. In a Secure Boot environment, you must either disable Secure Boot or sign the kernel yourself and register it with MOK. ([Elrepo][2])

Reboot.

```bash
sudo reboot
```

Select ELRepo’s `kernel-lt` in GRUB, then verify after booting.

```bash
uname -r
```

Confirm that the system has booted with a kernel version containing `elrepo`.

Also check whether the F2FS module can be used.

```bash
sudo modprobe f2fs
grep f2fs /proc/filesystems
```

If this fails, you should not proceed with converting the rootfs to F2FS.

---

## 5. Create a dracut Configuration to Include the F2FS Driver

Once the rootfs is F2FS, the F2FS driver is needed during the initramfs stage. Therefore, persist the `dracut` configuration.

```bash
sudo tee /etc/dracut.conf.d/90-f2fs-root.conf >/dev/null <<'EOF'
add_drivers+=" f2fs "
EOF
```

In `dracut.conf`, kernel modules specified in `add_drivers` can be added to the initramfs. Specify the module name without `.ko`. ([Ubuntu Manpages][3])

You may also use `force_drivers` if necessary.

```bash
sudo tee /etc/dracut.conf.d/90-f2fs-root.conf >/dev/null <<'EOF'
force_drivers+=" f2fs "
EOF
```

Like `add_drivers`, `force_drivers` includes the module, but it is a setting intended to have it `modprobe`d at an earlier stage. ([Ubuntu Manpages][3])

Rebuild the initramfs.

```bash
sudo dracut -f /boot/initramfs-$(uname -r).img $(uname -r)
```

Check its contents.

```bash
lsinitrd /boot/initramfs-$(uname -r).img | grep f2fs
```

It is sufficient if `f2fs.ko` is included.

---

## 6. Boot from a Live USB

From here, boot using an Ubuntu Live USB or similar and perform the work.

First, check the disk layout.

```bash
sudo lsblk -f
sudo blkid
```

Here, assume the following example:

```text
/dev/nvme0n1p1  /boot/efi
/dev/nvme0n1p2  /boot
/dev/nvme0n1p3  /
```

The rootfs is assumed to be `/dev/nvme0n1p3`.

---

## 7. Record the UUID of the rootfs

Under this policy, the same UUID as the original rootfs is specified when running `mkfs.f2fs`.

First, save the current UUID.

```bash
OLD_UUID=$(sudo blkid -s UUID -o value /dev/nvme0n1p3)
echo "$OLD_UUID"
```

Use this UUID when creating the F2FS filesystem as well.

---

## 8. Back Up the rootfs Before Conversion

Mount the rootfs before conversion.

```bash
sudo mkdir -p /mnt/src
sudo mount /dev/nvme0n1p3 /mnt/src
```

Mount the backup destination. Example:

```bash
sudo mkdir -p /mnt/backup
sudo mount /dev/sdX1 /mnt/backup
```

When backing up with `tar`, make sure not to lose SELinux contexts, xattrs, ACLs, or ownership information.

```bash
sudo tar \
  --one-file-system \
  --xattrs --xattrs-include='*' \
  --acls \
  --selinux \
  --numeric-owner \
  --sparse \
  -cpf /mnt/backup/rocky-root.tar \
  -C /mnt/src .
```

This backup is a logical backup of the rootfs.

As additional insurance, it is also a good idea to back up the entire pre-conversion partition using `partclone` or similar. A `partclone` backup serves as insurance for “restoring the original ext4/XFS rootfs if the conversion fails.”

After backing up, unmount it.

```bash
sudo umount /mnt/src
```

---

## 9. Create the rootfs as F2FS

Now recreate the rootfs partition as F2FS.

If your version of `mkfs.f2fs` supports specifying a UUID, specify the original UUID.

```bash
sudo mkfs.f2fs -f -l rocky-root -U "$OLD_UUID" /dev/nvme0n1p3
```

After creation, check whether the UUID is the same.

```bash
sudo blkid /dev/nvme0n1p3
```

Expected state:

```text
UUID="<OLD_UUID>" TYPE="f2fs"
```

Whether `mkfs.f2fs -U` can be used may depend on the version of `f2fs-tools`, so check in advance.

```bash
mkfs.f2fs -h
```

If you can keep the same UUID, you can avoid making major changes to `root=UUID=...` in BLS/GRUB. However, **you must change the filesystem type in fstab to F2FS**.

---

## 10. Restore the rootfs

Mount the rootfs that was converted to F2FS.

```bash
sudo mkdir -p /mnt/dst
sudo mount -t f2fs /dev/nvme0n1p3 /mnt/dst
```

Restore from the tar archive.

```bash
sudo tar \
  --xattrs --xattrs-include='*' \
  --acls \
  --selinux \
  --numeric-owner \
  --same-owner \
  --sparse \
  -xpf /mnt/backup/rocky-root.tar \
  -C /mnt/dst
```

Sync.

```bash
sudo sync
```

---

## 11. Edit `/etc/fstab`

Edit the restored fstab.

```bash
sudo vi /mnt/dst/etc/fstab
```

Change the rootfs line to F2FS.

Example:

```fstab
UUID=<OLD_UUID>  /  f2fs  defaults,noatime  0  0
```

Because the UUID is kept the same in this procedure, the UUID itself does not need to be changed.

However, this must be changed:

```text
ext4 or xfs → f2fs
```

If prioritizing recoverability, set the final pass number to `0` at first.

```fstab
UUID=<OLD_UUID>  /  f2fs  defaults,noatime  0  0
```

If you can reliably include `fsck.f2fs` in the initramfs and operate it that way, review this later.

---

## 12. Mount `/boot` and Check the BLS Entries

Since `/boot` is a separate partition, be sure to mount it.

```bash
sudo mount /dev/nvme0n1p2 /mnt/dst/boot
```

In a UEFI environment, also mount the EFI System Partition.

```bash
sudo mount /dev/nvme0n1p1 /mnt/dst/boot/efi
```

Check the BLS entries.

```bash
grep -R "root=" /mnt/dst/boot/loader/entries/*.conf
```

Since the UUID is kept the same, `root=UUID=...` can basically remain unchanged.

However, it is clearer to add `rootfstype=f2fs`.

Chroot and add it with `grubby`.

```bash
sudo mount --rbind /dev /mnt/dst/dev
sudo mount --make-rslave /mnt/dst/dev
sudo mount -t proc proc /mnt/dst/proc
sudo mount -t sysfs sysfs /mnt/dst/sys
sudo mount -t tmpfs tmpfs /mnt/dst/run

sudo chroot /mnt/dst /bin/bash
```

Run the following inside the chroot.

```bash
grubby --update-kernel=ALL --args="rootfstype=f2fs"
```

On RHEL 9 systems, boot entry kernel command lines can be changed with `grubby`. Red Hat documentation also shows `grubby --update-kernel=ALL --args=...` as a method for adding kernel parameters to all boot entries. ([Red Hat Documentation][4])

Check the result.

```bash
grep -R "root=" /boot/loader/entries/*.conf
grep -R "rootfstype" /boot/loader/entries/*.conf
```

---

## 13. Regenerate the initramfs Inside the chroot

When chrooting from a Live USB, **do not use `uname -r`**.

This is because `uname -r` returns the kernel version of the Live USB environment.

Check the Rocky-side kernel versions in `/lib/modules`.

```bash
ls /lib/modules
```

Target the ELRepo kernel.

```bash
KVER=$(ls /lib/modules | grep elrepo | tail -n 1)
echo "$KVER"
```

Prioritizing recoverability, first create a broader initramfs.

```bash
dracut -f \
  --no-hostonly \
  --force-drivers "f2fs" \
  "/boot/initramfs-${KVER}.img" \
  "${KVER}"
```

dracut is a tool for creating initramfs images; it collects the necessary tools/files from the installed system and builds the initramfs. ([GitHub][5])

Check it.

```bash
lsinitrd "/boot/initramfs-${KVER}.img" | grep f2fs
```

Confirm that `f2fs.ko` is included.

---

## 14. Schedule an SELinux Relabel

Even if xattrs and SELinux contexts were preserved with tar, it is safer to schedule a relabel on the first boot.

Inside the chroot:

```bash
touch /.autorelabel
```

A relabel will run after the first boot.

---

## 15. Exit the chroot and Reboot

Exit the chroot.

```bash
exit
```

Unmount.

```bash
sudo sync
sudo umount -R /mnt/dst
```

Reboot.

```bash
sudo reboot
```

In GRUB, select ELRepo’s `kernel-lt`.

---

## 16. Check After Booting

If the system boots successfully, check whether the rootfs is F2FS.

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /
```

Expected output:

```text
/dev/nvme0n1p3 f2fs ...
```

Also check the kernel.

```bash
uname -r
```

Confirm that it is the ELRepo kernel.

```bash
grep f2fs /proc/filesystems
```

SELinux can remain permissive for now.

```bash
getenforce
```

If there are no problems, switch it back to enforcing as needed.

```bash
sudo vi /etc/selinux/config
```

```ini
SELINUX=enforcing
```

Check again after rebooting.

```bash
getenforce
```

---

# Recovery Procedure If the System Does Not Boot

If the system does not boot after converting to F2FS, recover by chrooting from an Ubuntu Live USB.

## 1. Boot from a Live USB and Check

```bash
sudo lsblk -f
sudo blkid
```

Check whether the rootfs is recognized as F2FS.

```bash
sudo file -s /dev/nvme0n1p3
```

---

## 2. Mount the rootfs, boot, and EFI

```bash
sudo mkdir -p /mnt/sysroot
sudo mount -t f2fs /dev/nvme0n1p3 /mnt/sysroot
```

Mount `/boot`.

```bash
sudo mount /dev/nvme0n1p2 /mnt/sysroot/boot
```

For UEFI:

```bash
sudo mount /dev/nvme0n1p1 /mnt/sysroot/boot/efi
```

This is important.

If you run `dracut` without mounting `/boot`, the initramfs will be created in the empty directory inside the rootfs, not in the actual `/boot`.

---

## 3. chroot

```bash
sudo mount --rbind /dev /mnt/sysroot/dev
sudo mount --make-rslave /mnt/sysroot/dev
sudo mount -t proc proc /mnt/sysroot/proc
sudo mount -t sysfs sysfs /mnt/sysroot/sys
sudo mount -t tmpfs tmpfs /mnt/sysroot/run
```

```bash
sudo chroot /mnt/sysroot /bin/bash
```

---

## 4. Check fstab

```bash
cat /etc/fstab
```

Confirm that the rootfs is F2FS.

```fstab
UUID=<OLD_UUID>  /  f2fs  defaults,noatime  0  0
```

---

## 5. Check the BLS Entries

```bash
grep -R "root=" /boot/loader/entries/*.conf
grep -R "rootfstype" /boot/loader/entries/*.conf
```

If the UUID is kept the same, `root=UUID=...` can basically remain unchanged.

However, if `rootfstype=f2fs` is missing, add it.

```bash
grubby --update-kernel=ALL --args="rootfstype=f2fs"
```

---

## 6. Explicitly Specify the Target Kernel and Re-run dracut

Do not use `uname -r` here either.

```bash
ls /lib/modules
```

Select the ELRepo kernel.

```bash
KVER=$(ls /lib/modules | grep elrepo | tail -n 1)
echo "$KVER"
```

Rebuild the initramfs.

```bash
dracut -f \
  --no-hostonly \
  --force-drivers "f2fs" \
  "/boot/initramfs-${KVER}.img" \
  "${KVER}"
```

Check it.

```bash
lsinitrd "/boot/initramfs-${KVER}.img" | grep f2fs
```

---

## 7. Schedule an SELinux Relabel

```bash
touch /.autorelabel
```

---

## 8. Exit the chroot and Reboot

```bash
exit
```

```bash
sudo sync
sudo umount -R /mnt/sysroot
sudo reboot
```

Select ELRepo `kernel-lt` in GRUB.

---

# Common Causes of Failure

| Symptom                                        | Cause                                                                                              |
| ---------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `unknown filesystem type 'f2fs'`               | The F2FS module is not included in the initramfs                                                   |
| `VFS: Unable to mount root fs`                 | The rootfs specification, initramfs, or kernel module is incorrect                                 |
| `UUID=... does not exist`                      | The UUID changed, or the UUID was specified incorrectly                                            |
| Drops into emergency shell                     | fstab, root specification, initramfs, or SELinux issue                                             |
| Does not boot with standard kernel             | The Rocky 9 standard kernel side does not have the F2FS root module, or it is not in the initramfs |
| Still not fixed after running dracut in chroot | The initramfs was created without mounting `/boot`                                                 |
| `dracut -f $(uname -r)` fails                  | The Live USB kernel version is being used                                                          |

---

# Summary

The procedure for changing the rootfs of Rocky Linux 9 to F2FS is safer when done as follows.

```text
1. Install Rocky 9 normally with /boot as a separate partition
2. Set SELinux to permissive
3. Install ELRepo kernel-lt
4. Boot with kernel-lt and check the f2fs module
5. Persistently add f2fs to dracut
6. Boot from an Ubuntu Live USB
7. Back up the rootfs with tar, including xattrs/ACLs/SELinux
8. Record the original rootfs UUID
9. Specify the original UUID when running mkfs.f2fs
10. Restore the rootfs
11. Change the rootfs type in /etc/fstab to f2fs
12. Add rootfstype=f2fs to the BLS entries
13. Explicitly specify the target kernel inside the chroot and regenerate dracut
14. Create /.autorelabel
15. Boot with ELRepo kernel-lt
```

The following four points are especially important.

```text
- Always keep /boot separate
- Include f2fs.ko in the initramfs
- Confirm that the UUID remains the same after mkfs.f2fs
- If the system does not boot, recover using Live USB + chroot + dracut
```

If you keep the UUID the same as before, you can avoid changing `root=UUID=...`, which makes the work considerably simpler. However, the filesystem type in `/etc/fstab` and the F2FS module in the initramfs must be configured correctly.

[1]: https://elrepo.org/wiki/doku.php?id=kernel-lt "kernel-lt [ELRepo Wiki]"
[2]: https://elrepo.org/wiki/doku.php?id=secureboot "secureboot [ELRepo Wiki]"
[3]: https://manpages.ubuntu.com/manpages/xenial/man5/dracut.conf.5.html "dracut.conf - configuration file(s) for dracut"
[4]: https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_monitoring_and_updating_the_kernel/configuring-kernel-command-line-parameters_managing-monitoring-and-updating-the-kernel "Chapter 4. Configuring kernel command-line parameters"
[5]: https://github.com/dracutdevs/dracut "dracut the event driven initramfs infrastructure"
