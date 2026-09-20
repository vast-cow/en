---
title: "How to Install UEK and bcache-tools on Rocky Linux 9"
description: "Strategy   Keep Rocky Linux 9 BaseOS/AppStream unchanged, and only retrieve the following..."
pubDatetime: 2026-08-02T06:11:33.033Z
---

## Strategy

Keep Rocky Linux 9 BaseOS/AppStream unchanged, and only retrieve the following from the Oracle Linux side:

* UEK itself: `kernel-uek` and its dependent subpackages
* `bcache-tools`
* Do not install Oracle Linux userland, `oraclelinux-release-el9`, Oracle's `glibc`, etc.

The Oracle Linux Yum Server officially provides instructions for use from RHEL-compatible distributions, but the configuration of running UEK on Rocky Linux is not considered officially supported by either Oracle or Rocky. ([Oracle Linux Yum Server][1])

The following assumes **x86_64**.

## UEK R7 or R8

As of August 2026, the following are available for Oracle Linux 9:

| Series | Kernel Series | Selection Criteria |
| ------ | ------------: | ----------------- |
| UEK R7 |          5.15 | Relatively conservative in mixed configuration with Rocky 9 |
| UEK R8 |          6.12 | Prioritize newer hardware/features |

Oracle's current UEK R8 repository contains 6.12 series `kernel-uek`, `kernel-uek-core`, and various modules packages. ([Oracle Linux Yum Server][2])
UEK R7 is 5.15 series. ([Oracle Linux Yum Server][3])

Here, to slightly reduce mixed configuration risk, we'll use **UEK R7** as an example. To switch to R8, replace `UEKR7` with `UEKR8` in the URLs.

---

## 1. Pre-check

```bash
cat /etc/rocky-release
uname -m
findmnt /boot
findmnt /boot/efi 2>/dev/null || true
mokutil --sb-state 2>/dev/null || true
```

Do not remove the existing Rocky kernel. It serves as recovery if UEK fails to boot.

Before working, update Rocky to its normal state.

```bash
sudo dnf upgrade --refresh
sudo reboot
```

After reboot:

```bash
uname -r
```

---

## 2. Register Oracle Linux 9 Signing Key

Place Oracle's official OL9 key.

```bash
sudo curl -fsSL \
  https://yum.oracle.com/RPM-GPG-KEY-oracle-ol9 \
  -o /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
```

Verify the fingerprint.

```bash
gpg --show-keys --with-fingerprint \
  /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
```

Confirm it matches at least the following fingerprints published by Oracle:

```text
3E6D 826D 3FBA B389 C2F3 8E34 BC4D 06A0 8D8B 756F
9822 3175 9C74 6706 5D0C E9B2 A7DD 0708 8B4E FBE6
```

The OL9 key acquisition source and fingerprints are published by Oracle. ([Oracle Linux Yum Server][4])

Also import into the RPM database.

```bash
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
```

---

## 3. Create a Minimally Restricted Oracle Repository

Do not install `oraclelinux-release-el9`; create your own repo file with just two entries.

```bash
sudo tee /etc/yum.repos.d/oracle-uek-minimal.repo >/dev/null <<'EOF'
[oracle-uek-r7-minimal]
name=Oracle Linux 9 UEK R7 - restricted
baseurl=https://yum.oracle.com/repo/OracleLinux/OL9/UEKR7/$basearch/
enabled=0
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
includepkgs=kernel-uek,kernel-uek-core,kernel-uek-modules,kernel-uek-modules-extra
metadata_expire=6h
skip_if_unavailable=0

[oracle-baseos-bcache-minimal]
name=Oracle Linux 9 BaseOS - bcache-tools only
baseurl=https://yum.oracle.com/repo/OracleLinux/OL9/baseos/latest/$basearch/
enabled=0
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
includepkgs=bcache-tools
metadata_expire=6h
skip_if_unavailable=0
EOF
```

Two important points:

* `enabled=0`: Normal `dnf upgrade` won't use Oracle repositories
* `includepkgs=`: Prevents packages other than explicitly listed ones from being obtained from Oracle

The Oracle Linux 9 BaseOS URL is the same as what Oracle recommends for RHEL-compatible environments. ([Oracle Linux Yum Server][1])

### If Using UEK R8

Instead of R7, use the following UEK entry:

```ini
[oracle-uek-r8-minimal]
name=Oracle Linux 9 UEK R8 - restricted
baseurl=https://yum.oracle.com/repo/OracleLinux/OL9/UEKR8/$basearch/
enabled=0
gpgcheck=1
repo_gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-oracle-ol9
includepkgs=kernel-uek,kernel-uek-core,kernel-uek-modules-core,kernel-uek-modules,kernel-uek-modules-extra
metadata_expire=6h
skip_if_unavailable=0
```

R8 has different package splitting from R7, including `kernel-uek-modules-core`. ([Oracle Linux Yum Server][2])

---

## 4. Verify Packages Visible from Oracle

First, check candidates without installing.

```bash
sudo dnf clean metadata

sudo dnf \
  --disablerepo='oracle-*' \
  --enablerepo=oracle-uek-r7-minimal \
  repoquery --available 'kernel-uek*'
```

Also check `bcache-tools`.

```bash
sudo dnf \
  --disablerepo='oracle-*' \
  --enablerepo=oracle-baseos-bcache-minimal \
  repoquery --available --info bcache-tools
```

On Oracle Linux 9, `bcache-tools` is provided as an Oracle BaseOS additional package. ([Oracle Docs][5])

### Verify Sources

```bash
sudo dnf repoquery \
  --available \
  --qf '%{name}-%{evr}.%{arch} <- %{repoid}' \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  kernel-uek bcache-tools
```

---

## 5. Pre-check the Transaction

First, use `--assumeno`.

```bash
sudo dnf install --assumeno \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  kernel-uek bcache-tools
```

Check these points:

* Only `kernel-uek*` and `bcache-tools` come from Oracle
* `glibc`, `systemd`, `dracut`, `grub2`, etc. are not replaced with Oracle versions
* Rocky BaseOS/AppStream packages are not removed
* `--allowerasing` is not required

Abort if any unexpected Oracle packages appear.

For a stricter check:

```bash
sudo dnf install --assumeno -v \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  kernel-uek bcache-tools
```

---

## 6. Install UEK and bcache-tools

If the pre-check passes, execute.

```bash
sudo dnf install \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  kernel-uek bcache-tools
```

The `kernel-uek` meta package installs `kernel-uek-core` and modules as dependencies. The UEK R7 repository contains these at the same version. ([Oracle Linux Yum Server][3])

Verify:

```bash
rpm -qa | grep -E '^(kernel-uek|bcache-tools)' | sort
```

Also verify the vendor:

```bash
rpm -q \
  --qf '%{NAME} %{VERSION}-%{RELEASE} | %{VENDOR}\n' \
  kernel-uek bcache-tools
```

Check all Oracle-origin packages:

```bash
rpm -qa \
  --qf '%{NAME} %{VERSION}-%{RELEASE} | %{VENDOR}\n' |
grep -i oracle |
sort
```

Confirm there are no unintended Oracle userland packages.

---

## 7. Verify initramfs and bcache Module

Installing UEK usually generates initramfs.

List installed UEK:

```bash
rpm -q kernel-uek-core
ls -lh /boot/vmlinuz-*uek /boot/initramfs-*uek.img
```

Check if bcache module exists in the UEK kernel.

```bash
UEK_VER="$(rpm -q --qf '%{VERSION}-%{RELEASE}.%{ARCH}\n' \
  kernel-uek-core | sort -V | tail -1)"

echo "$UEK_VER"
modinfo -k "$UEK_VER" bcache
```

If `modinfo` returns information, the UEK has the bcache module.

To explicitly include it in initramfs:

```bash
sudo dracut --force \
  --add-drivers bcache \
  "/boot/initramfs-${UEK_VER}.img" \
  "$UEK_VER"
```

However, if the root filesystem is not on bcache, forcing inclusion in initramfs at boot is usually unnecessary. You can `modprobe bcache` after boot.

---

## 8. Verify Registration in GRUB

```bash
sudo grubby --info=ALL |
grep -E '^(index|kernel|title)='
```

Identify the UEK entry.

```bash
sudo grubby --info=ALL |
grep -B2 -A3 'el9uek'
```

It's safer not to set UEK as default initially; instead, select UEK from the GRUB menu once for booting.

To make the GRUB menu easier to display:

```bash
sudo grub2-editenv - unset menu_auto_hide
```

---

## 9. Test Boot with UEK

After rebooting, select the entry containing `el9uek` from GRUB.

```bash
sudo reboot
```

After booting:

```bash
uname -r
```

Expected example:

```text
5.15.0-...el9uek.x86_64
```

Check bcache:

```bash
sudo modprobe bcache
lsmod | grep '^bcache'
```

Check tools:

```bash
make-bcache --version
bcache-super-show --help
```

Also check kernel logs.

```bash
sudo journalctl -b -k -p warning
sudo dmesg -T | grep -iE 'bcache|error|failed|firmware'
```

Verify network, storage, console, and SELinux.

```bash
ip addr
findmnt
getenforce
systemctl --failed
```

---

## 10. Make UEK Default if No Issues

To make the currently booting UEK the default:

```bash
sudo grubby --set-default "/boot/vmlinuz-$(uname -r)"
sudo grubby --default-kernel
```

Or, specify the latest installed UEK:

```bash
UEK_KERNEL="$(ls -1 /boot/vmlinuz-*el9uek* | sort -V | tail -1)"
sudo grubby --set-default "$UEK_KERNEL"
sudo grubby --default-kernel
```

Keep the Rocky standard kernel.

```bash
rpm -q kernel-core
ls -1 /boot/vmlinuz-*
```

---

## 11. Update Method

Oracle repositories remain disabled, so normal updates target only Rocky.

```bash
sudo dnf upgrade
```

Only enable Oracle repositories explicitly when updating UEK and `bcache-tools`.

```bash
sudo dnf upgrade \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  'kernel-uek*' bcache-tools
```

To be more cautious, pre-check each time.

```bash
sudo dnf upgrade --assumeno \
  --enablerepo=oracle-uek-r7-minimal \
  --enablerepo=oracle-baseos-bcache-minimal \
  'kernel-uek*' bcache-tools
```

### Avoid `dnf upgrade --enablerepo=oracle-...` Alone

It's safer not to run unspecified updates like:

```bash
# Not recommended
sudo dnf upgrade --enablerepo=oracle-uek-r7-minimal
```

Currently restricted by `includepkgs`, but explicitly specifying update targets prevents accidents.

---

## 12. Secure Boot Considerations

When Secure Boot is enabled, the main issue is not RPM signatures but whether **Rocky's shim/firmware trusts Oracle's kernel signature**.

Check:

```bash
mokutil --sb-state
```

If `SecureBoot enabled`, UEK may fail to boot with errors like:

```text
Verification failed
Security Violation
Bad shim signature
```

In this mixed configuration, it's practical to test with one of the following:

1. Disable Secure Boot during verification
2. Properly register the Oracle kernel signing certificate in MOK

The latter requires certificate acquisition, verification, and MOK enrollment, which is not resolved by simply registering the Oracle RPM GPG key. RPM package signing keys and UEFI Secure Boot kernel signing certificates are different.

---

## 13. Minimum Precautions Before Using bcache

`make-bcache` destroys existing data on target devices. Verify device names thoroughly.

```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,TYPE,FSTYPE,MOUNTPOINTS
```

Example:

```bash
# SSD cache side
sudo make-bcache --cache /dev/nvme0n1p1

# HDD backend side
sudo make-bcache --bdev /dev/sdb
```

This writes the bcache superblock to the specified devices. Before substituting actual device names, ensure backups and console access.

According to Linux kernel documentation, bcache supports writethrough and writeback, with writeback disabled by default. ([Linux Kernel Documentation][6])

For initial testing, using writethrough is more appropriate to minimize write loss risk.

---

## Rollback

If UEK fails to boot, select the Rocky standard kernel from GRUB.

Return the Rocky kernel to default:

```bash
ROCKY_KERNEL="$(ls -1 /boot/vmlinuz-*el9_* 2>/dev/null |
  grep -v el9uek |
  sort -V |
  tail -1)"

sudo grubby --set-default "$ROCKY_KERNEL"
sudo grubby --default-kernel
```

Remove UEK:

```bash
sudo dnf remove 'kernel-uek*'
```

If `bcache-tools` is also unnecessary:

```bash
sudo dnf remove bcache-tools
```

Disable or remove the repo file:

```bash
sudo mv \
  /etc/yum.repos.d/oracle-uek-minimal.repo \
  /etc/yum.repos.d/oracle-uek-minimal.repo.disabled
```

## Recommended Final Configuration

* Rocky BaseOS/AppStream: always enabled
* Oracle UEK repo: `enabled=0`
* Oracle BaseOS repo: `enabled=0`
* Oracle side `includepkgs`:

  * `kernel-uek`
  * `kernel-uek-core`
  * Modules packages required for the UEK series
  * `bcache-tools`
* Rocky standard kernel: always keep at least one generation
* UEK updates: manually execute by specifying package names
* Secure Boot: pre-verification required
* DKMS/kmod products: individually verify UEK ABI compatibility

This method effectively limits Oracle-origin packages to the kernel set and `bcache-tools`, preventing Oracle-ization of the Rocky userland.
