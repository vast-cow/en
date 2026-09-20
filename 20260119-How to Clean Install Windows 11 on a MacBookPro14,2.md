---
title: "How to Clean Install Windows 11 on a MacBookPro14,2"
description: "This article explains the steps required to perform a clean installation of Windows 11 on a..."
pubDatetime: 2026-01-19T11:26:42.074Z
---

This article explains the steps required to perform a clean installation of Windows 11 on a **MacBookPro14,2**, while ensuring proper hardware functionality such as the Touch Bar.

## Prerequisites

* A MacBookPro14,2 with macOS already installed
* A Windows 11 installation USB
* An external storage device (USB flash drive or external SSD)
* Boot Camp drivers downloaded in advance

## Step 1: Download Boot Camp Drivers

While macOS is still installed on the Mac, download the required Boot Camp drivers.

1. Launch the **Boot Camp Assistant**.
2. Use it to download the Windows support software (drivers).
3. Save the downloaded drivers to an external storage device such as a USB flash drive.

Alternatively, you can download the Boot Camp drivers using the following tool:

* [https://github.com/timsutton/brigadier](https://github.com/timsutton/brigadier)

This method is useful if Boot Camp Assistant cannot retrieve the drivers directly.

## Step 2: Start the Windows Installer

1. Boot the Mac from the Windows 11 installation USB.
2. When the Windows installer starts, proceed until you reach the disk selection screen.

## Step 3: Delete Existing Partitions (With Caution)

On the disk selection screen:

1. Delete all partitions **except the ESP (EFI System Partition)**.
2. Inside the ESP, **do not delete the `APPLE` directory**.

> **Important:**
> The `APPLE` directory inside the ESP is essential for proper Touch Bar functionality. Removing it may cause the Touch Bar to stop working in Windows.

## Step 4: Install Windows 11

1. Select the remaining unallocated space.
2. Proceed with the Windows 11 installation.
3. Wait for the installation to complete and for Windows to boot successfully.

## Step 5: Install Boot Camp Drivers

After Windows 11 has finished installing:

1. Connect the external storage device containing the Boot Camp drivers.
2. Run the Boot Camp installer (`Setup.exe`).
3. Follow the on-screen instructions to install all drivers.
4. Reboot Windows when prompted.

## Conclusion

By carefully preserving the ESP and the `APPLE` directory while performing a clean installation, you can successfully install Windows 11 on a MacBookPro14,2 with full hardware support, including the Touch Bar. Installing the correct Boot Camp drivers afterward ensures stable and usable Windows performance on Apple hardware.
