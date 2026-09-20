---
title: "ELRepo Kernel Configuration with Grubby"
description: "You can configure it as follows. Whether using kernel-ml or kernel-lt, ELRepo kernels are typically..."
pubDatetime: 2026-06-11T11:41:58.676Z
---

You can configure it as follows. Whether using `kernel-ml` or `kernel-lt`, ELRepo kernels are typically visible as `/boot/vmlinuz-*-elrepo.*`.

## 1. Check the ELRepo kernel

```bash
sudo grubby --info=ALL | awk 'BEGIN{RS=""} /elrepo/ {print $0 "\n"}'
```

Or for a quick check:

```bash
ls -1 /boot/vmlinuz-*elrepo*
```

## 2. Set the latest ELRepo kernel as the default

```bash
ELREPO_KERNEL=$(ls -1 /boot/vmlinuz-*elrepo* | sort -V | tail -n 1)

sudo grubby --set-default "$ELREPO_KERNEL"
```

The official Red Hat documentation also describes using `grubby --set-default /boot/vmlinuz-...` to permanently change the default kernel. ([Red Hat Documentation][1])

## 3. Verify the configuration

```bash
grubby --default-kernel
grubby --default-index
```

The expected result is that `--default-kernel` points to the ELRepo kernel.

Example:

```bash
/boot/vmlinuz-6.x.x-x.el9.elrepo.x86_64
```

ELRepo provides `kernel-ml` as the mainline stable series and `kernel-lt` as the long-term support series. ([elrepo.org][2])

## 4. Verify after reboot

```bash
sudo reboot
```

After booting:

```bash
uname -r
```

If the kernel version contains `elrepo`, the configuration was successful.

```bash
6.x.x-x.el9.elrepo.x86_64
```

## If you want to select only `kernel-ml`

```bash
ELREPO_KERNEL=$(rpm -q kernel-ml --qf '/boot/vmlinuz-%{VERSION}-%{RELEASE}.%{ARCH}\n' | sort -V | tail -n 1)

sudo grubby --set-default "$ELREPO_KERNEL"
grubby --default-kernel
```

## If you want to select only `kernel-lt`

```bash
ELREPO_KERNEL=$(rpm -q kernel-lt --qf '/boot/vmlinuz-%{VERSION}-%{RELEASE}.%{ARCH}\n' | sort -V | tail -n 1)

sudo grubby --set-default "$ELREPO_KERNEL"
grubby --default-kernel
```

[1]: https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/setting-a-kernel-as-default_assembly_the-linux-kernel "1.7. Setting a Kernel as the Default"
[2]: https://elrepo.org/wiki/doku.php?id=kernel-ml "Kernel-ml"
