---
title: "Create a macOS Installer USB Without a Mac"
description: "Download an Apple recovery image with macrecovery.py, convert it with dmg2img, and write it to USB from Linux or Windows, with boot and imaging troubleshooting notes."
pubDatetime: 2026-01-20T02:18:49.507Z
---

If you don’t have access to a Mac, you can still prepare a bootable macOS installer USB from Linux or Windows. Below is a simple, practical workflow that uses OpenCore’s `macrecovery.py` to fetch Apple’s installer files, converts the downloaded `.dmg` to a raw image, and writes it to a USB drive.

## What you’ll need

* A PC running Linux **or** Windows
* A USB drive (16 GB or larger recommended)
* Python 3 (for running `macrecovery.py`)
* `dmg2img` (Linux packages often provide it; on Windows you can use WSL or a prebuilt binary)
* A disk writer:

  * Linux: `dd`
  * Windows: **Rufus**



## 1) Download the macOS recovery image with `macrecovery.py`

OpenCore provides a small Python script that talks to Apple’s servers and downloads the official recovery image.

**Get the script:**

* From the OpenCorePkg repo, locate `Utilities/macrecovery/macrecovery.py`.
* Download the file (or clone the repo) to a local folder, e.g., `~/macrecovery/`.

**Example (Linux/macOS shell or Windows PowerShell/WSL):**

```bash
cd ~/macrecovery
python3 macrecovery.py -h
```

**Download a specific version (examples):**

```bash
# Latest recovery for Intel Macs:
python3 macrecovery.py --latest --catalog 10.15  # or another catalog key

# Or specify product codes (see script help / repo docs)
# Example generic form:
python3 macrecovery.py -b Mac-XXXXX -m 00000000000000000 download
```

When it completes, you should have a file like `RecoveryImage.dmg` (name may vary) in the working directory.

> Tip: If you’re not sure which flags to use, start with `--latest` and check the script’s `-h` help for options.



## 2) Convert the DMG to a raw image (`.img`)

Use `dmg2img` to convert Apple’s DMG into a raw disk image that standard imaging tools can write.

**Linux (or WSL):**

```bash
sudo apt-get update && sudo apt-get install -y dmg2img   # Debian/Ubuntu
dmg2img RecoveryImage.dmg RecoveryImage.img
```

If your downloaded file has a different name, adjust the command accordingly.



## 3) Write the image to a USB drive

### Option A: Linux with `dd`

1. Insert your USB and identify its device path:

   ```bash
   lsblk
   ```

   Suppose it shows up as `/dev/sdX` (replace **X** with the actual letter).

2. Write the image (this will erase the USB):

   ```bash
   sudo dd if=RecoveryImage.img of=/dev/sdX bs=4M status=progress conv=fsync
   ```

3. Safely eject:

   ```bash
   sync
   sudo eject /dev/sdX
   ```

### Option B: Windows with Rufus

1. Download and run **Rufus**.
2. Select your USB drive.
3. Click **SELECT** and choose `RecoveryImage.img`.
4. Use the default settings unless you know you need something specific.
5. Click **START** and wait until the write completes.
6. Safely remove the USB.



## 4) Boot from the installer USB

* Insert the USB into the target machine.
* Power it on and choose the USB as the boot device (use your motherboard’s boot menu key, e.g., F12/F8/F11/ESC).
* Follow the on-screen prompts to access macOS Recovery/Installer tools.



## Notes & Troubleshooting

* **USB size**: 16 GB is usually safe; some images fit on 8 GB, but give yourself headroom.
* **Write errors**: If `dd` fails, ensure the USB isn’t mounted and that you’re using the correct device path (not a partition like `/dev/sdX1`).
* **Rufus quirks:** If Rufus complains about the image format, ensure you used `dmg2img` and are selecting the resulting `.img`, not the original `.dmg`.
* **Hardware compatibility**: Downloaded recovery images are official Apple resources; installing macOS on non-Apple hardware may require additional steps and may not be supported.

That’s it—download with `macrecovery.py`, convert with `dmg2img`, and write with `dd` or Rufus.
