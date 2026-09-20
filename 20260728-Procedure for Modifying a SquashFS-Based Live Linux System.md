---
title: "Procedure for Modifying a SquashFS-Based Live Linux System"
description: "A Live Linux system such as SystemRescue generally has the following structure:    ISO9660 ├── EFI/,..."
pubDatetime: 2026-07-28T06:53:42.170Z
---

A Live Linux system such as SystemRescue generally has the following structure:

```text
ISO9660
├── EFI/, boot/, syslinux/, grub/   ← Bootloader
├── vmlinuz                         ← Kernel
├── initramfs                       ← Initial RAM disk
└── airootfs.sfs / filesystem.squashfs
    └── Actual root filesystem
```

Because SquashFS is read-only, the basic process is as follows:

```text
Extract the ISO
  ↓
Extract the SquashFS
  ↓
Edit the rootfs or enter it with chroot
  ↓
Rebuild the SquashFS
  ↓
Replace the SquashFS inside the ISO
  ↓
Rebuild it as a bootable ISO
  ↓
Test with BIOS and UEFI
```

However, **with SystemRescue, it is safer not to rebuild `airootfs.sfs` directly from the outset, but to select a method in the following order of priority:**

1. YAML configuration in `sysrescue.d`
2. Overlay using an SRM (SystemRescueModule)
3. Direct reconstruction of `airootfs.sfs`
4. Full build from the SystemRescue source

The official SystemRescue documentation also recommends `sysrescue-customize` for modifying ISO images. An SRM is an additional layer in SquashFS format, and files at the same paths in the SRM take precedence over those in the base rootfs. ([SystemRescue][1])

---

## 1. Preparing the Working Environment

It is easiest to perform this work on Linux. On Debian/Ubuntu-based systems, install the following:

```bash
sudo apt update
sudo apt install squashfs-tools xorriso rsync file
```

It is also useful to install QEMU for testing:

```bash
sudo apt install qemu-system-x86 ovmf
```

The official SystemRescue customization script also lists `xorriso` and `squashfs-tools` among its main dependencies. It can also be run under WSL. ([SystemRescue][1])

Create a working directory:

```bash
mkdir -p ~/work/systemrescue
cd ~/work/systemrescue

cp /path/to/systemrescue.iso original.iso
```

Ensure that you have at least several times the original ISO size in free space. When rebuilding from within SystemRescue itself, the official documentation notes that the Copy-on-Write area may require approximately three times the ISO size. ([SystemRescue][1])

---

# Method A: Use the Official SystemRescue `sysrescue-customize` Tool

For SystemRescue, this is the appropriate first choice.

Download `sysrescue-customize` from the official page and make it executable:

```bash
chmod +x sysrescue-customize
sudo install -m 755 sysrescue-customize /usr/local/bin/
```

## Extract the ISO

```bash
sysrescue-customize \
  --unpack \
  --source="$PWD/original.iso" \
  --dest="$PWD/iso-tree"
```

After extraction, the structure will look something like this:

```bash
find iso-tree -maxdepth 3 -type f | sort | less
```

Do not assume that the SquashFS location is fixed across versions. Search for it:

```bash
find iso-tree -type f \
  \( -name '*.sfs' -o -name '*.squashfs' -o -name '*.sqfs' \) \
  -print
```

On SystemRescue, the target is normally `airootfs.sfs`.

## When Modifying Only ISO-Level Files

Files such as boot configuration, YAML configuration, and autorun scripts that do not need to reside inside the rootfs can be added directly to the ISO tree.

Example:

```bash
sudo mkdir -p iso-tree/sysrescue.d

sudo tee iso-tree/sysrescue.d/500-local.yaml >/dev/null <<'EOF'
sysconfig:
  keyboard: jp
EOF
```

Match the available YAML keys to the official configuration specification for the relevant version. YAML files in `sysrescue.d` are merged in lexical order, and settings on the boot command line ultimately take precedence. ([SystemRescue][2])

Rebuild the ISO:

```bash
sysrescue-customize \
  --rebuild \
  --source="$PWD/iso-tree" \
  --dest="$PWD/systemrescue-custom.iso"
```

To overwrite an existing file:

```bash
sysrescue-customize \
  --rebuild \
  --source="$PWD/iso-tree" \
  --dest="$PWD/systemrescue-custom.iso" \
  --overwrite
```

The official script is designed to preserve the ISO structure captured during extraction while rebuilding it, making this safer than manually reproducing the El Torito and UEFI boot configuration. ([SystemRescue][1])

---

# Method B: Overlay Files onto the rootfs with an SRM

For configuration files, scripts, static binaries, and small additional packages, it is appropriate to place them in an SRM instead of modifying the base `airootfs.sfs`.

An SRM is also a SquashFS image, but it is layered over the base rootfs with OverlayFS during boot. A file can also be replaced by placing another file at the same path. ([SystemRescue][3])

## Create the SRM Directory

The directory structure should match the rootfs as it will appear after boot:

```bash
mkdir -p srm-root/usr/local/bin
mkdir -p srm-root/etc/systemd/system
mkdir -p srm-root/root
```

For example, add a custom script:

```bash
cat >srm-root/usr/local/bin/local-rescue-tool <<'EOF'
#!/bin/sh
echo "Custom SystemRescue tool"
EOF

chmod 755 srm-root/usr/local/bin/local-rescue-tool
```

To override a configuration file, place it at the same path:

```bash
mkdir -p srm-root/etc
cp my-config.conf srm-root/etc/my-config.conf
```

## Include the SRM in the ISO

```bash
sysrescue-customize \
  --rebuild \
  --source="$PWD/iso-tree" \
  --dest="$PWD/systemrescue-srm.iso" \
  --srm-dir="$PWD/srm-root"
```

When `--srm-dir` is specified, the script compresses that directory into an SRM and adds configuration to the ISO to enable the SRM. Note that this process also modifies the ISO tree specified by `--source`. ([SystemRescue][1])

## Specify Ownership and Permissions

For example, explicitly specify modes when adding an SSH public key:

```bash
cat >srm-root/.squashfs-pseudo <<'EOF'
/root/.ssh m 700 root root
/root/.ssh/authorized_keys m 600 root root
EOF
```

As a general rule, avoid embedding private keys in an ISO.

## Specify Compression Options

You can create `.squashfs-options` directly under the SRM root:

```bash
cat >srm-root/.squashfs-options <<'EOF'
-comp zstd
-b 1M
EOF
```

Because the contents of this file are evaluated as shell code, you must not run a recipe obtained from a third party without reviewing it. The official documentation also warns that it can lead to arbitrary command execution. ([SystemRescue][1])

---

# Method C: Directly Extract and Modify `airootfs.sfs`

This procedure is for cases where the base rootfs itself must be modified.

The following assumes that the ISO has already been extracted into `iso-tree`.

## 1. Identify the Target SquashFS

```bash
find iso-tree -type f \
  \( -name 'airootfs.sfs' \
     -o -name 'filesystem.squashfs' \
     -o -name '*.sqfs' \) \
  -print
```

Store it in a variable:

```bash
SFS="$(find iso-tree -type f -name 'airootfs.sfs' -print -quit)"

test -n "$SFS" || {
  echo "airootfs.sfs was not found" >&2
  exit 1
}

printf 'Target: %s\n' "$SFS"
```

## 2. Record the Original SquashFS Information

When rebuilding the image, match the original compression method and block size as closely as possible:

```bash
unsquashfs -s "$SFS" | tee squashfs-original.txt
```

The main items to check are:

```text
Compression
Block size
Filesystem size
Xattrs
Fragments
Duplicates
```

SquashFS can use compression methods such as gzip, xz, lzo, lz4, and zstd. The block size and compression method affect boot speed, memory usage, and ISO size. ([Ubuntu Manpages][4])

## 3. Extract the SquashFS

Ensure that the destination directory does not already exist:

```bash
sudo rm -rf rootfs
sudo unsquashfs -d rootfs "$SFS"
```

Verify the extracted contents:

```bash
sudo ls -la rootfs
sudo cat rootfs/etc/os-release
```

To preserve file ownership, device nodes, extended attributes, and similar metadata, extraction and reconstruction should generally be performed with root privileges.

---

## 4. Simple File Modifications

If you only need to add configuration files or scripts, chroot is unnecessary:

```bash
sudo install -Dm755 local-rescue-tool \
  rootfs/usr/local/bin/local-rescue-tool
```

Add a configuration file:

```bash
sudo install -Dm644 my-config.conf \
  rootfs/etc/my-config.conf
```

Example of adding a systemd unit:

```bash
sudo install -Dm644 my-service.service \
  rootfs/etc/systemd/system/my-service.service
```

You can enable it with a symbolic link:

```bash
sudo mkdir -p rootfs/etc/systemd/system/multi-user.target.wants

sudo ln -sfn ../my-service.service \
  rootfs/etc/systemd/system/multi-user.target.wants/my-service.service
```

However, match the link to the unit’s `WantedBy=` setting and the actual target structure.

---

## 5. Prepare the chroot Environment

Use chroot when installing packages, creating users, or executing commands.

### Mount the Virtual Filesystems

```bash
sudo mount --rbind /dev rootfs/dev
sudo mount --make-rslave rootfs/dev

sudo mount -t proc proc rootfs/proc
sudo mount --rbind /sys rootfs/sys
sudo mount --make-rslave rootfs/sys

sudo mount --rbind /run rootfs/run
sudo mount --make-rslave rootfs/run
```

If DNS resolution is required, temporarily replace `resolv.conf`. Because the original may be a symbolic link, record its state first:

```bash
sudo ls -l rootfs/etc/resolv.conf
sudo cp -a rootfs/etc/resolv.conf \
  rootfs/etc/resolv.conf.before-chroot 2>/dev/null || true

sudo cp -L /etc/resolv.conf rootfs/etc/resolv.conf
```

### Enter the chroot

```bash
sudo chroot rootfs /bin/bash
```

Inside the chroot:

```bash
export HOME=/root
export LC_ALL=C

source /etc/os-release
printf '%s %s\n' "$ID" "$VERSION_ID"
```

---

## 6. Add Packages

### SystemRescue / Arch Linux-Based Systems

SystemRescue is based on Arch Linux, and additional packages can be installed using `pacman`. ([SystemRescue][5])

For example:

```bash
pacman -Sy --needed tmux
```

However, avoid a full upgrade such as:

```bash
# Do not run this as a general rule
pacman -Syu
```

The reason is that the kernel and initramfs are normally located outside the SquashFS:

```text
vmlinuz inside the ISO             Remains old
initramfs inside the ISO           Remains old
modules inside airootfs.sfs        Become new
userspace inside airootfs.sfs      Becomes new
```

This can cause inconsistencies such as:

* `/usr/lib/modules/<kernel-version>` no longer matching
* Kernel modules failing to load
* Tools inside the initramfs no longer matching libraries in the rootfs
* Updating only glibc or systemd and making the system unbootable
* Breaking pacman snapshot configuration

The official SystemRescue documentation also warns that SRMs created for a different version, or SRMs that replace core libraries, can cause instability or prevent the system from booting. ([SystemRescue][3])

### Debian/Ubuntu-Based Live ISOs

```bash
apt-get update
apt-get install --no-install-recommends tmux
```

Remove unnecessary data:

```bash
apt-get clean
rm -rf /var/lib/apt/lists/*
```

Here as well, if you update the kernel package, you must also update the external `vmlinuz` and initramfs.

---

## 7. Post-Processing Inside the chroot

Delete temporary files and logs:

```bash
rm -rf /tmp/*
rm -rf /var/tmp/*
rm -f /root/.bash_history
```

Check how the original image handles `machine-id`:

```bash
ls -l /etc/machine-id
cat /etc/machine-id
```

If the original was an empty file, restore it to an empty file:

```bash
: >/etc/machine-id
```

Exit:

```bash
exit
```

---

## 8. Unmount the Filesystems

Unmount them in reverse order:

```bash
sudo umount -R rootfs/run 2>/dev/null || true
sudo umount -R rootfs/sys 2>/dev/null || true
sudo umount -R rootfs/proc 2>/dev/null || true
sudo umount -R rootfs/dev 2>/dev/null || true
```

Verify that no mounts remain:

```bash
findmnt -R "$(realpath rootfs)"
```

If nothing is displayed, everything has been unmounted.

If you temporarily modified `resolv.conf`, restore it:

```bash
if sudo test -e rootfs/etc/resolv.conf.before-chroot ||
   sudo test -L rootfs/etc/resolv.conf.before-chroot; then
  sudo rm -f rootfs/etc/resolv.conf
  sudo mv rootfs/etc/resolv.conf.before-chroot \
    rootfs/etc/resolv.conf
fi
```

---

## 9. Rebuild the SquashFS

Match the original `unsquashfs -s` results.

For example, suppose the original showed:

```text
Compression zstd
Block size 1048576
```

Rebuild it:

```bash
sudo rm -f airootfs.sfs.new

sudo mksquashfs rootfs airootfs.sfs.new \
  -noappend \
  -comp zstd \
  -b 1M
```

For xz:

```bash
sudo mksquashfs rootfs airootfs.sfs.new \
  -noappend \
  -comp xz \
  -b 1M
```

Check the rebuilt image:

```bash
unsquashfs -s airootfs.sfs.new
```

Extract part of it for verification:

```bash
rm -rf verify-root
unsquashfs -d verify-root airootfs.sfs.new \
  usr/local/bin/local-rescue-tool

ls -l verify-root/usr/local/bin/local-rescue-tool
```

## Common Recompression Mistakes

### Using `-all-root` Without Careful Consideration

```bash
mksquashfs rootfs output.sfs -all-root
```

This makes every file root-owned, including files that should belong to ordinary users. Depending on the Live environment, this can break user home directories and service files.

### Using a Different Compression Method from the Original

If the SquashFS implementation available in the initramfs does not support the selected compression method, it will be unable to mount the image. Be especially careful with ISOs intended for older kernels.

### Losing Extended Attributes

In environments that use Linux capabilities or SELinux attributes, losing xattrs may prevent commands from functioning correctly. `mksquashfs` normally preserves xattrs by default, so do not specify `-no-xattrs`.

---

## 10. Replace the Original SquashFS

Keep a backup:

```bash
sudo mv "$SFS" "${SFS}.original"
sudo install -m 644 airootfs.sfs.new "$SFS"
```

Compare them:

```bash
ls -lh "$SFS" "${SFS}.original"
```

---

# Rebuilding the ISO

## For SystemRescue

Use the official script:

```bash
sysrescue-customize \
  --rebuild \
  --source="$PWD/iso-tree" \
  --dest="$PWD/systemrescue-custom-rootfs.iso"
```

This is the safest method.

---

## When Replacing Only the SquashFS in a Generic ISO

Instead of rebuilding the entire ISO from scratch with `mkisofs`, loading the original ISO with `xorriso` and replacing only the target file makes it easier to preserve the boot information.

Determine the path of the SquashFS inside the ISO:

```bash
ISO_SFS_PATH="/${SFS#iso-tree/}"

printf '%s\n' "$ISO_SFS_PATH"
```

For example:

```text
/sysresccd/x86_64/airootfs.sfs
```

Replace it and generate a new ISO:

```bash
rm -f custom.iso

xorriso \
  -indev original.iso \
  -outdev custom.iso \
  -map airootfs.sfs.new "$ISO_SFS_PATH" \
  -boot_image any replay
```

To add another file at the same time:

```bash
xorriso \
  -indev original.iso \
  -outdev custom.iso \
  -map airootfs.sfs.new "$ISO_SFS_PATH" \
  -map local.cfg /config/local.cfg \
  -boot_image any replay
```

Example of removing a file:

```bash
xorriso \
  -indev original.iso \
  -outdev custom.iso \
  -rm /path/in/iso/unneeded-file \
  -map airootfs.sfs.new "$ISO_SFS_PATH" \
  -boot_image any replay
```

For complex hybrid ISOs, GRUB2 MBR configurations, and ISOs with additional partitions, prioritize distribution-specific build tools. `xorriso` boot replay reuses the original ISO’s boot settings, but version-dependent issues have also been reported when replacing the boot image itself. ([Scdbackup][6])

---

# Handling Checksums

Some Live ISOs contain integrity information such as:

```text
md5sum.txt
SHA256SUMS
Embedded MD5
Checksums used with copytoram
```

Once the rootfs is modified, these values will naturally no longer match.

SystemRescue has embedded checksums in its ISO images since earlier releases, and version 13.01 added the `checksum` boot option for integrity checking when using `copytoram`. This is one reason to use the official reconstruction tool. ([SystemRescue][7])

For a generic ISO, if a file such as `md5sum.txt` exists, regenerate it:

```bash
cd iso-tree

find . -type f \
  ! -name md5sum.txt \
  -print0 |
  sort -z |
  xargs -0 md5sum |
  sudo tee md5sum.txt >/dev/null

cd ..
```

However, checksum formats differ between distributions, so inspect the format of the existing file first.

---

# Functional Testing

## 1. Check the ISO Structure

```bash
file custom.iso
xorriso -indev custom.iso -toc
xorriso -indev custom.iso -report_el_torito plain
```

## 2. Check the SquashFS Inside the ISO

Extract it temporarily:

```bash
rm -f verify.sfs

xorriso \
  -osirrox on \
  -indev custom.iso \
  -extract "$ISO_SFS_PATH" verify.sfs
```

Check it:

```bash
unsquashfs -s verify.sfs
```

Verify that the intended file is present:

```bash
rm -rf verify-root
unsquashfs -d verify-root verify.sfs \
  usr/local/bin/local-rescue-tool

ls -l verify-root/usr/local/bin/local-rescue-tool
```

## 3. Test BIOS Boot with QEMU

```bash
qemu-system-x86_64 \
  -m 4096 \
  -smp 2 \
  -cdrom custom.iso \
  -boot d
```

If KVM is available:

```bash
qemu-system-x86_64 \
  -enable-kvm \
  -cpu host \
  -m 4096 \
  -smp 2 \
  -cdrom custom.iso \
  -boot d
```

## 4. Test UEFI Boot

The location of the OVMF files varies by distribution:

```bash
find /usr/share -iname 'OVMF_CODE*.fd' -o -iname 'OVMF_VARS*.fd'
```

Example:

```bash
cp /usr/share/OVMF/OVMF_VARS_4M.fd OVMF_VARS.fd

qemu-system-x86_64 \
  -enable-kvm \
  -m 4096 \
  -smp 2 \
  -drive if=pflash,format=raw,readonly=on,file=/usr/share/OVMF/OVMF_CODE_4M.fd \
  -drive if=pflash,format=raw,file="$PWD/OVMF_VARS.fd" \
  -cdrom custom.iso
```

At minimum, test the following:

* Whether it boots in Legacy BIOS mode
* Whether it boots in UEFI mode
* Whether the SquashFS mounts successfully
* Whether systemd finishes booting
* Whether networking works
* Whether the added commands can be executed
* Whether the existing storage, encryption, and RAID tools still work
* Whether it also boots with `copytoram`

---

# Automation Recipe

With `sysrescue-customize --auto`, SystemRescue modifications can be represented as code using the following structure:

```text
recipe/
├── iso_delete/
├── iso_add/
├── iso_patch_and_script/
└── build_into_srm/
```

* `iso_delete`: Delete files from the ISO
* `iso_add`: Add files to or overwrite files in the ISO
* `iso_patch_and_script`: Apply patches and execute arbitrary scripts
* `build_into_srm`: Rootfs overlay to be built into an SRM

The official script processes these in this order. Scripts are executed with the root of the ISO tree as the current directory, so a recipe can also extract and rebuild `airootfs.sfs`. ([SystemRescue][1])

Example:

```text
recipe/
├── iso_add/
│   └── sysrescue.d/
│       └── 500-local.yaml
└── build_into_srm/
    └── usr/
        └── local/
            └── bin/
                └── local-rescue-tool
```

Run it:

```bash
sysrescue-customize \
  --auto \
  --source="$PWD/original.iso" \
  --dest="$PWD/custom.iso" \
  --recipe-dir="$PWD/recipe" \
  --work-dir="$PWD/auto-work"
```

Keeping the recipe under Git version control makes it easier to reapply the modifications to newer SystemRescue releases. The official documentation also describes auto mode as a mechanism for applying the same changes to different versions. ([SystemRescue][1])

---

# When Modifying the Kernel or initramfs as Well

The following modifications are not well suited to a method that only rebuilds the SquashFS:

* Kernel updates
* Kernel configuration changes
* Adding drivers
* Adding initramfs hooks
* Major GRUB/Syslinux changes
* Replacing large numbers of packages
* Updating foundational packages such as glibc, systemd, or pacman

In these cases, use the SystemRescue source build.

SystemRescue is based on Arch Linux and builds its official ISO using a patched version of Archiso. The source code, build tools, and documentation are also publicly available. ([SystemRescue][8])

An Archiso profile generally manages the following:

```text
profile/
├── airootfs/
├── efiboot/
├── syslinux/
├── grub/
├── packages.x86_64
├── pacman.conf
└── profiledef.sh
```

Use `packages.x86_64` to select packages, `airootfs/` to manage files added to the rootfs, and `profiledef.sh` to specify the SquashFS compression method and boot modes. ([GitHub][9])

If you intend to maintain a custom edition continuously, this approach is more appropriate than repeatedly disassembling a prebuilt ISO.

---

# Secure Boot

In general, modifying signed EFI binaries, kernels, Unified Kernel Images, or similar components invalidates their signatures. You must sign them again with your own key and register that key in the firmware.

Regarding SystemRescue, an official forum administrator stated on January 22, 2026, that Secure Boot was not supported by default. The changelog for version 13.01, released on June 6, 2026, also contains no mention of added Secure Boot support, so it is best not to assume that the standard image can boot with Secure Boot enabled. ([System-Rescue][10])

---

# Criteria for Selecting a Method

| Modification                                   | Recommended method             |
| ---------------------------------------------- | ------------------------------ |
| Keyboard, network, and autorun configuration   | `sysrescue.d` YAML             |
| Adding scripts or static binaries              | SRM                            |
| Adding a small number of packages              | SRM or `cowpacman2srm`         |
| Replacing files under `/etc`                   | SRM                            |
| Completely removing files from the base rootfs | Direct SquashFS reconstruction |
| Large-scale package changes                    | Source/Archiso build           |
| Kernel or module changes                       | Source/Archiso build           |
| initramfs changes                              | Source/Archiso build           |
| One-time experiments                           | Direct SquashFS reconstruction |
| Repeated application to newer versions         | `sysrescue-customize --auto`   |

In practice, the most maintainable separation is: **use YAML for configuration, SRMs for additions, and a source build for major modifications that include the kernel.**

[1]: https://www.system-rescue.org/scripts/sysrescue-customize/ "SystemRescue - sysrescue-customize: customize SystemRescue ISO-images"
[2]: https://www.system-rescue.org/manual/Configuring_SystemRescue/ "SystemRescue - Configuring SystemRescue"
[3]: https://www.system-rescue.org/Modules/ "SystemRescue - Custom SystemRescue Modules"
[4]: https://manpages.ubuntu.com/manpages/stonking/man1/unsquashfs.1.html?utm_source=chatgpt.com "unsquashfs - tool to uncompress, extract and list squashfs ..."
[5]: https://www.system-rescue.org/manual/Installing_packages_with_pacman/?utm_source=chatgpt.com "Installing additional software packages with pacman"
[6]: https://scdbackup.sourceforge.net/xorriso_eng.html?utm_source=chatgpt.com "GNU xorriso - GNU Project - Free Software Foundation"
[7]: https://www.system-rescue.org/Changes-x86/ "SystemRescue - ChangeLog"
[8]: https://www.system-rescue.org/manual/Building_SystemRescue_from_Source/ "SystemRescue - Building SystemRescue from Source"
[9]: https://github.com/archlinux/archiso/blob/master/docs/README.profile.rst "archiso/docs/README.profile.rst at master · archlinux/archiso · GitHub"
[10]: https://forums.system-rescue.org/t/cannot-boot-into-systemrescue-with-secure-boot-enabled/42?utm_source=chatgpt.com "Cannot boot into SystemRescue with Secure boot enabled"
