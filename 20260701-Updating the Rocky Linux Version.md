---
title: "Updating the Rocky Linux Version"
description: "Upgrading from Rocky Linux 9.5 to 9.6 can normally be done with a standard DNF update.          ..."
pubDatetime: 2026-07-01T04:42:29.972Z
---

Upgrading from Rocky Linux 9.5 to 9.6 can normally be done with a standard DNF update.

## Check the Current Version

```bash
cat /etc/os-release
```

or

```bash
cat /etc/rocky-release
```

## Update Package Metadata

First, refresh the repository metadata.

```bash
sudo dnf clean all
sudo dnf makecache
```

## Update the Entire System

```bash
sudo dnf upgrade --refresh
```

or

```bash
sudo dnf update --refresh
```

Rocky Linux does not lock minor releases (9.5 → 9.6); it updates to the latest available 9.x release.

## Reboot

If the kernel or systemd has been updated, reboot the system.

```bash
sudo reboot
```

## Verify the Version

After rebooting, check the version.

```bash
cat /etc/rocky-release
```

Expected output:

```text
Rocky Linux release 9.6 (Blue Onyx)
```

---

## ELevate and Leapp Are Not Required

Since 9.5 → 9.6 is an update within the same major version,

* Leapp
* ELevate

are not required.

These tools are used for major version upgrades such as 8.x → 9.x or 9.x → 10.x.

---

## If You Have Pinned a Specific Minor Version

You can check this with:

```bash
sudo dnf config-manager --dump | grep releasever
```

or

```bash
cat /etc/dnf/vars/releasever
```

If `9.5` is set, either remove it or change it to `9`.

```bash
sudo rm -f /etc/dnf/vars/releasever
```

Then run:

```bash
sudo dnf upgrade --refresh
```
