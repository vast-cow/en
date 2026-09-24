---
title: "Keeping Rocky Linux 9 Up to Date with `dnf-automatic`"
description: "Configure DNF timers for automatic updates or MOTD-only notifications, control package downloads, and verify which timer settings take precedence."
pubDatetime: 2026-08-04T06:50:08.333Z
updatedDate: 2026-08-04T06:51:09.476Z
---

Keeping your system updated is one of the most important steps for maintaining security, stability, and performance. On **Rocky Linux 9.6**, the `dnf-automatic` package can periodically check for available updates and either notify administrators, download packages, or install updates automatically.

This guide covers installation, configuration, automatic updates, and a notification-only configuration that uses MOTD without downloading or installing packages.

---

## 1. Install `dnf-automatic`

The `dnf-automatic` tool is not installed by default. Install it with:

```bash
sudo dnf install -y dnf-automatic
```

---

## 2. Configure `dnf-automatic`

The main configuration file is:

```text
/etc/dnf/automatic.conf
```

Open it with your preferred text editor:

```bash
sudo nano /etc/dnf/automatic.conf
```

### Key options

#### Upgrade type

The `upgrade_type` setting controls which updates are detected:

```ini
[commands]
upgrade_type = security
```

Available values include:

- `default` — all available updates
- `security` — only updates associated with security advisories

#### Download updates

To allow automatic package downloads:

```ini
download_updates = yes
```

To prevent automatic package downloads:

```ini
download_updates = no
```

#### Apply updates automatically

To install available updates automatically:

```ini
apply_updates = yes
```

To prevent automatic installation:

```ini
apply_updates = no
```

When `apply_updates = yes` is configured, packages must also be downloaded.

#### Emitters

The `[emitters]` section controls how results are reported. Available mechanisms include standard output, email, custom commands, and MOTD.

For example, to report results through MOTD:

```ini
[emitters]
emit_via = motd
```

The MOTD emitter writes the update report to `/etc/motd`, allowing administrators to see it when logging in through SSH or a local console.

---

## 3. Enable Automatic Updates

The generic `dnf-automatic.timer` follows the behavior configured in `/etc/dnf/automatic.conf`.

For example, to download and install security updates automatically, configure:

```ini
[commands]
upgrade_type = security
download_updates = yes
apply_updates = yes
```

Then enable and start the timer:

```bash
sudo systemctl enable --now dnf-automatic.timer
```

This causes `dnf-automatic` to check for and process updates according to the configuration file.

---

## 4. Notification Only with MOTD

To notify administrators about available updates without downloading or installing any packages, configure `dnf-automatic` to use the MOTD emitter.

Edit `/etc/dnf/automatic.conf`:

```bash
sudo nano /etc/dnf/automatic.conf
```

Use the following settings:

```ini
[commands]
upgrade_type = security
download_updates = no
apply_updates = no

[emitters]
emit_via = motd
```

This configuration:

- Checks repository metadata for available security updates
- Does not download update packages
- Does not install update packages
- Writes the result to `/etc/motd`

Enable the notification-only timer:

```bash
sudo systemctl enable --now dnf-automatic-notifyonly.timer
```

The `dnf-automatic-notifyonly.timer` unit overrides the download and installation behavior and performs notification only. The explicit `download_updates = no` and `apply_updates = no` settings are retained in the configuration for clarity and as a safe default when the generic timer is used.

### Disable download and installation timers

To ensure that no other `dnf-automatic` timer downloads or installs packages, disable the generic, download, and installation timers:

```bash
sudo systemctl disable --now dnf-automatic.timer
sudo systemctl disable --now dnf-automatic-download.timer
sudo systemctl disable --now dnf-automatic-install.timer
```

Then enable only the notification timer:

```bash
sudo systemctl enable --now dnf-automatic-notifyonly.timer
```

Verify the enabled timers:

```bash
systemctl list-timers --all | grep dnf-automatic
```

Only `dnf-automatic-notifyonly.timer` should be enabled for the notification-only configuration.

### Check the MOTD output

Run the service manually to test the configuration:

```bash
sudo systemctl start dnf-automatic-notifyonly.service
```

Display the generated MOTD:

```bash
cat /etc/motd
```

The message will also be displayed during the next SSH or console login, provided the system's login configuration displays `/etc/motd`.

> **Note:** Checking for updates still requires refreshing or reading repository metadata. The notification-only configuration prevents RPM packages from being downloaded; it does not eliminate repository metadata traffic.

---

## 5. Other Timer Modes

`dnf-automatic` provides several systemd timers for different operating modes.

### Follow `/etc/dnf/automatic.conf`

```bash
sudo systemctl enable --now dnf-automatic.timer
```

### Notify only

Checks for updates and reports the result without downloading or installing packages:

```bash
sudo systemctl enable --now dnf-automatic-notifyonly.timer
```

### Download without installation

Downloads available packages but does not install them:

```bash
sudo systemctl enable --now dnf-automatic-download.timer
```

### Download and install

Downloads and installs available updates:

```bash
sudo systemctl enable --now dnf-automatic-install.timer
```

The specialized timer units override the corresponding download and installation settings in `/etc/dnf/automatic.conf`.

---

## 6. Verification and Troubleshooting

Check the notification-only timer:

```bash
systemctl status dnf-automatic-notifyonly.timer
```

Check when it will run next:

```bash
systemctl list-timers --all | grep dnf-automatic
```

Check the service logs:

```bash
journalctl -u dnf-automatic-notifyonly.service
```

Check recent logs for all `dnf-automatic` units:

```bash
journalctl -u 'dnf-automatic*'
```

Confirm which timers are enabled:

```bash
systemctl is-enabled dnf-automatic.timer
systemctl is-enabled dnf-automatic-notifyonly.timer
systemctl is-enabled dnf-automatic-download.timer
systemctl is-enabled dnf-automatic-install.timer
```

For a notification-only system, the expected result is:

```text
disabled
enabled
disabled
disabled
```

---

## Conclusion

Rocky Linux 9.6 can use `dnf-automatic` in several modes:

- Automatic installation for systems that should remain patched without manual intervention
- Automatic download for systems where installation is performed separately
- Notification only for controlled environments where administrators review updates before taking action

For a notification-only configuration with no package downloads, use:

```ini
[commands]
upgrade_type = security
download_updates = no
apply_updates = no

[emitters]
emit_via = motd
```

Enable only:

```bash
sudo systemctl enable --now dnf-automatic-notifyonly.timer
```

This configuration reports available security updates through MOTD while leaving package download and installation under manual control.
