---
title: "PiKVM Setup and Basic Usage"
description: "PiKVM is a remote KVM solution that lets you view video output, control the keyboard and mouse, and..."
pubDatetime: 2026-07-30T09:34:26.288Z
updatedDate: 2026-08-31T13:21:24.555Z
---

PiKVM is a remote KVM solution that lets you view video output, control the keyboard and mouse, and boot from virtual media even when the target computer's operating system is not running.

This guide walks through the entire process in the order you would actually perform it, from writing the OS image to an SD card, through initial configuration, virtual media setup, remote access with Tailscale, and finally operating the target machine.

Keep in mind that the interface and some features vary depending on the PiKVM model and KVMD version. The virtual media sections in this article primarily assume PiKVM V2 or later.

---

## Understanding the Read-Only Filesystem

PiKVM OS normally operates with its filesystem mounted as read-only to reduce SD card wear and protect against filesystem corruption caused by unexpected power loss. After making configuration changes, you should always return the filesystem to read-only mode.

An important point is that PiKVM provides **two different commands for enabling writes**, each serving a different purpose.

### Comparison of Write Mode Commands

| Command                                | Target                                    | Primary Purpose                                                                            | Enable Writes                   | Return to Read-Only             |
| -------------------------------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------- | ------------------------------- |
| `rw` / `ro`                            | PiKVM OS root filesystem                  | Changing passwords, installing packages, editing files under `/etc`, changing the hostname | `rw`                            | `ro`                            |
| `kvmd-helper-otgmsd-remount rw` / `ro` | Virtual media area at `/var/lib/kvmd/msd` | Uploading, deleting, or modifying ISO and IMG files                                        | `kvmd-helper-otgmsd-remount rw` | `kvmd-helper-otgmsd-remount ro` |

The key point is that **running `rw` does not necessarily make the virtual media area writable**. Likewise, running `kvmd-helper-otgmsd-remount rw` does **not** allow you to modify system configuration files such as those under `/etc`.

Use the appropriate command depending on what you are modifying.

---

# 1. Write the Image to the SD Card

First, write the appropriate PiKVM OS image for your PiKVM model to an SD card.

After writing completes—but before booting PiKVM—open the first FAT32 partition on the SD card. This partition contains the `pikvm.txt` file used for first-boot configuration.

---

# 2. Configure Wi-Fi Before the First Boot

If wired Ethernet is unavailable, configuring Wi-Fi before the first boot makes the initial setup much easier.

Add the following lines to `pikvm.txt`:

```text
WIFI_ESSID='Your Wi-Fi SSID'
WIFI_PASSWD='Your Wi-Fi Password'
WIFI_REGDOM='JP'
```

`WIFI_REGDOM='JP'` specifies Japan as the wireless regulatory domain.

If the following line already exists in `pikvm.txt` immediately after writing the image, leave it unchanged:

```text
FIRST_BOOT=1
```

`FIRST_BOOT=1` instructs PiKVM to generate SSH host keys, certificates, and other first-boot resources.

If your Wi-Fi password contains a backslash, it must be escaped by doubling it:

```text
WIFI_PASSWD='abc\\def'
```

After the configuration has been applied, `pikvm.txt` is automatically deleted.

Also note that configurations based on the Raspberry Pi Zero 2 W cannot use 5 GHz Wi-Fi.

Safely eject the SD card, insert it into the PiKVM, and power it on.

---

# 3. Find the PiKVM IP Address

PiKVM normally obtains an IP address using DHCP.

After booting, the assigned IP address is displayed on the monitor connected to PiKVM, making it the easiest way to find it.

If no monitor is attached, check your router's management interface or DHCP lease table to identify the assigned address.

Throughout this guide, the PiKVM IP address will be represented as:

```text
192.168.1.100
```

---

# 4. Connect to PiKVM via SSH

By default, the Linux administrator account is configured as follows:

| Item     | Default |
| -------- | ------- |
| Username | `root`  |
| Password | `root`  |

Connect using an SSH client:

```bash
ssh root@192.168.1.100
```

On the first connection, SSH asks you to verify the host key. Confirm that you are connecting to the correct device before accepting it.

PiKVM also has a separate `admin` account for the Web UI. By default, both the username and password are `admin`.

The Linux `root` account and the Web UI `admin` account are completely independent, so both passwords must be changed.

---

# 5. Change Both Default Passwords First

Running PiKVM with its default passwords is insecure. Change them before configuring networking or installing additional features.

First, remount the root filesystem as writable:

```bash
rw
```

Change the Linux `root` password:

```bash
passwd root
```

Next, change the Web UI `admin` password:

```bash
kvmd-htpasswd set admin
```

After both passwords have been updated, return the filesystem to read-only mode:

```bash
ro
```

The complete sequence is:

```bash
rw
passwd root
kvmd-htpasswd set admin
ro
```

If you forget to run `rw` before changing passwords, the changes cannot be saved.

The official documentation also recommends changing both passwords between `rw` and `ro`.

---

# 6. Change the Hostname

If you manage multiple PiKVM devices, assigning meaningful hostnames based on their location or purpose makes administration easier.

For example, a PiKVM managing the first server in a server room might use:

```text
pikvm-server01
```

Change the hostname using:

```bash
rw
systemctl restart systemd-hostnamed
hostnamectl set-hostname pikvm-server01
ro
reboot
```

After rebooting, both the SSH prompt and the network hostname reflect the new name.

The official FAQ also recommends restarting `systemd-hostnamed`, then running `hostnamectl set-hostname`, followed by rebooting PiKVM.

If you plan to install Tailscale, configuring the hostname first makes devices easier to identify in the Tailscale admin console.

---

# 7. Update PiKVM OS

After the initial configuration is complete, update PiKVM OS.

```bash
pikvm-update
```

Normally you do **not** need to manually run `rw` or `ro`; `pikvm-update` is the recommended update method.

However, updates are never completely risk-free. The official documentation recommends performing updates while you still have physical access to the PiKVM, in case the SD card must be reflashed. Avoid updating a PiKVM that is only reachable over a VPN unless necessary.

If you're using an older image that doesn't include `pikvm-update`, install it first:

```bash
rw
pacman -Syy
pacman -S pikvm-os-updater
pikvm-update
```

Reboot afterward if required.

---

# 8. Install Tailscale on PiKVM

If you intend to access PiKVM over the Internet, using a private networking solution such as Tailscale is much safer and easier to manage than exposing the Web UI or SSH directly.

PiKVM provides a dedicated package called `tailscale-pikvm`.

After running `pikvm-update`, install it in the following order:

```bash
rw
pacman -S tailscale-pikvm
systemctl enable --now tailscaled
tailscale up --qr
ro
```

Running `tailscale up --qr` displays a QR code for the authentication URL directly in the terminal. This is especially convenient when using a remote console where copying long URLs is difficult. `--qr` is an officially supported option in the current Tailscale CLI.


### Using PiKVM as an Exit Node

If you want to use PiKVM as an Exit Node, enable the Exit Node feature in Tailscale.

```bash
rw

tailscale set \
  --advertise-exit-node \
  --advertise-routes=192.168.0.0/24

ro
```

For `--advertise-routes`, specify the LAN subnet that is reachable from the PiKVM. For example, if your managed network is `192.168.0.0/24`, use the command above.

After applying the configuration, open the device in the Tailscale admin console and approve both **Exit Node** and **Subnet routes**.

You can then select the PiKVM as the Exit Node from your Tailscale client. In addition to routing Internet traffic through the PiKVM, you'll also be able to access devices on the LAN connected to the PiKVM.

### Configuring UDP GRO on Raspberry Pi

When using a Raspberry Pi as an Exit Node, it is recommended to enable the Linux kernel's UDP GRO settings.

On PiKVM, you can create the following systemd service to apply the required settings automatically at boot.

```bash
rw

cat > /etc/systemd/system/apply-udp-gro.service <<'EOF'
[Unit]
Description=Configure UDP GRO forwarding
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now apply-udp-gro.service

ro
```

This service runs the following command each time the PiKVM boots, ensuring that the UDP GRO settings persist across reboots.

```bash
ethtool -K eth0 rx-udp-gro-forwarding on rx-gro-list off
```

After configuring the service, you can verify the current settings with:

```bash
ethtool -k eth0 | grep -E 'rx-udp-gro-forwarding|rx-gro-list'
```

The expected output is:

```text
rx-gro-list: off
rx-udp-gro-forwarding: on
```

If your PiKVM uses a network interface other than `eth0`, replace it with the appropriate interface name for your environment.


---

# 9. Upload Virtual Media to PiKVM

PiKVM can present ISO and IMG files to the target machine as either a virtual CD/DVD drive or a virtual USB flash drive.

Image files are stored in:

```text
/var/lib/kvmd/msd
```

Uploading through the Web UI's **Drive** menu is usually the simplest approach.

If you prefer using SCP or rsync manually, first remount the virtual media partition as writable:

```bash
kvmd-helper-otgmsd-remount rw
```

Transfer the ISO from your local computer:

```bash
scp installer.iso root@192.168.1.100:/var/lib/kvmd/msd/
```

After the transfer completes, remount the partition as read-only:

```bash
kvmd-helper-otgmsd-remount ro
```

The official documentation recommends the following sequence:

1. Run `kvmd-helper-otgmsd-remount rw`
2. Copy files into `/var/lib/kvmd/msd`
3. Run `kvmd-helper-otgmsd-remount ro`

Keeping the virtual media partition read-only during normal operation helps prevent corruption caused by unexpected power loss.

---

## Is `chown kvmd:kvmd` Necessary for ISO Files?

Files uploaded via SCP are normally owned by `root`.

```bash
ls -l /var/lib/kvmd/msd/
```

For example:

```text
-rw-r--r-- 1 root root ... installer.iso
```

As long as the file is readable by everyone, changing ownership is **not necessarily required** for read-only ISO usage.

In other words, you do **not** need to mechanically run:

```bash
chown kvmd:kvmd installer.iso
```

for every uploaded ISO.

What actually matters is that the `kvmd` user has permission to read the file. If necessary, set the permissions like this:

```bash
kvmd-helper-otgmsd-remount rw
chmod 644 /var/lib/kvmd/msd/installer.iso
kvmd-helper-otgmsd-remount ro
```

On the other hand, if the image will be presented as a writable virtual flash drive, write permissions are also required. The official documentation uses:

```bash
chmod 666 /var/lib/kvmd/msd/flash.img
```

for writable flash images.

If you prefer consistent ownership across all files, you may use:

```bash
kvmd-helper-otgmsd-remount rw
chown kvmd:kvmd /var/lib/kvmd/msd/installer.iso
chmod 644 /var/lib/kvmd/msd/installer.iso
kvmd-helper-otgmsd-remount ro
```

However, for read-only ISOs, the essential requirement is that the `kvmd` user can read the file—not that the file is owned by `kvmd`.

---

# 10. Boot the Target Machine from Virtual Media

After uploading an ISO, open the PiKVM Web UI and select the desired image from the **Drive** menu.

The basic workflow is:

1. Open the **Drive** menu.
2. Select the ISO or IMG file.
3. Choose the media type.
4. Connect the image.
5. Reboot the target machine.
6. Select the virtual media from the BIOS or UEFI boot menu.

PiKVM can present virtual media in the following formats:

| Mode   | Appears as      | Typical Use                                             |
| ------ | --------------- | ------------------------------------------------------- |
| CD/DVD | Optical drive   | Standard OS installation ISOs and rescue media          |
| Flash  | USB flash drive | UEFI compatibility workarounds and writable disk images |

In most cases, ISO images should be connected as **CD/DVD**.

However, some UEFI implementations cannot boot correctly from virtual optical drives. If that happens, reconnecting the same image as **Flash** may allow the firmware to recognize it.

This is a compatibility workaround for specific systems, not a universal requirement.

PiKVM virtual media is accessible from both BIOS and UEFI, and the connection mode can be switched between CD/DVD and Flash through the Web UI.

---

## Precautions When Using Virtual Media

Do not power off PiKVM while an image is being uploaded or while a writable virtual image is connected to the target machine.

Interrupting power during an upload or write operation can corrupt the image file or the filesystem.

---

# 11. Send Ctrl+Alt+Del

Since web browsers often intercept keys such as **Ctrl** and **Alt**, special key combinations should be sent from the PiKVM Web UI.

To send **Ctrl+Alt+Del**, open the **Shortcuts** menu at the top of the Web UI and select **Ctrl+Alt+Del**.

If the desired shortcut is unavailable, you can also use PiKVM's Magic Key shortcut mechanism. According to the current official documentation, you press and release the Magic Key, then press and release **Ctrl**, **Alt**, and **Del** in sequence.

---

# 12. Install Tailscale on the Target Machine

If you also want to install Tailscale on the computer you're controlling through PiKVM, the following command is particularly convenient on Linux:

```bash
tailscale up --qr
```

Scan the QR code displayed on the screen with your smartphone to complete Tailscale authentication.

This eliminates the need to manually type long authentication URLs or rely on clipboard sharing, making it especially useful immediately after installing an operating system or in environments without a graphical desktop.

---

# Recommended Setup Order

To minimize rework, perform the initial setup in the following order:

1. Write the PiKVM OS image to the SD card.
2. Configure Wi-Fi in `pikvm.txt` before the first boot.
3. Boot PiKVM and determine its IP address.
4. Connect via SSH as `root`.
5. Change the Linux `root` password.
6. Change the Web UI `admin` password.
7. Change the hostname.
8. Run `pikvm-update` while physical access is available.
9. Install Tailscale if needed.
10. Upload ISO and IMG files to the virtual media storage.
11. Connect to the target machine through the Web UI.
12. Attach virtual media and perform OS installation or recovery.

---

# Summary

The most common source of confusion when using PiKVM is switching between the two write modes.

When modifying system configuration, use:

```bash
rw
# Make configuration changes
ro
```

When uploading ISO or IMG files, use:

```bash
kvmd-helper-otgmsd-remount rw
# Upload or remove ISO/IMG files
kvmd-helper-otgmsd-remount ro
```

Understanding the distinction between these two mechanisms and always returning to read-only mode afterward is fundamental to operating PiKVM safely.

Also, immediately after the first boot, be sure to change **both** the Linux and Web UI passwords. If you need remote access from outside your local network, it is generally better to use a private networking solution such as Tailscale rather than exposing the PiKVM Web UI directly to the Internet.
