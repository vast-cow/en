---
title: "How to Disable Suspend on Lid Close in Rocky Linux"
description: "Configure systemd-logind to ignore laptop lid events on Rocky Linux, verify the settings, and troubleshoot desktop power handlers or suspend targets."
pubDatetime: 2026-06-21T06:51:22.992Z
---

In Rocky Linux, lid-close handling is generally managed by **systemd-logind**, even when using `multi-user.target`. Set `HandleLidSwitch=ignore`. The Rocky Linux documentation also recommends setting `HandleLidSwitch` to `ignore` in `/etc/systemd/logind.conf`. ([Rocky Linux Docs][1])

## Recommended Configuration

```bash
sudo cp -a /etc/systemd/logind.conf /etc/systemd/logind.conf.bak.$(date +%F-%H%M%S)

sudo mkdir -p /etc/systemd/logind.conf.d

sudo tee /etc/systemd/logind.conf.d/99-ignore-lid.conf >/dev/null <<'EOF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF

sudo systemctl restart systemd-logind.service
```

This ignores lid-close events in all cases: normal operation, when connected to AC power, and when docked. The default value of `HandleLidSwitch` is `suspend`; changing it to `ignore` prevents logind from suspending the system when the lid is closed. ([man7.org][2])

## Verification

```bash
systemd-analyze cat-config systemd/logind.conf | grep -E 'HandleLidSwitch'
```

Expected output:

```text
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

To check the current default target:

```bash
systemctl get-default
```

If you want to keep the system on `multi-user.target`:

```bash
sudo systemctl set-default multi-user.target
```

## If Editing `/etc/systemd/logind.conf` Directly

If drop-in configuration files are not supported in your environment, you can edit the main configuration file directly:

```bash
sudo vi /etc/systemd/logind.conf
```

Add the following under the `[Login]` section:

```ini
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

Apply the changes:

```bash
sudo systemctl restart systemd-logind.service
```

## If the System Still Suspends

First, inspect the logs:

```bash
journalctl -u systemd-logind -b | grep -i -E 'lid|suspend|sleep'
```

Then check whether another process is managing power events:

```bash
systemd-inhibit --list
```

With `multi-user.target`, desktop environments typically do not interfere. However, if a graphical session such as GNOME is running, the desktop environment may take over suspend handling. The systemd documentation notes that `Handle*` settings may be bypassed when another application holds a low-level inhibitor lock. ([man7.org][2])

As a last resort, you can disable suspend-related targets entirely:

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

To restore them later:

```bash
sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

In most cases, **setting `HandleLidSwitch=ignore` in logind is sufficient**. If you are using a laptop as a server and plan to operate it under heavy load with the lid closed, be aware that some models have reduced cooling performance when closed. The Red Hat documentation provides a similar caution. ([Red Hat Documentation][3])

[1]: https://docs.rockylinux.org/10/gemstones/scripts/NoSleep/ "NoSleep.sh - A simple Configuration Script - Documentation"
[2]: https://man7.org/linux/man-pages/man5/logind.conf.5.html "logind.conf(5) - Linux manual page"
[3]: https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/7/html/desktop_migration_and_administration_guide/closing-lid "13.10. Preventing a Laptop from Suspending When the Lid Is Closed | Desktop Migration and Administration Guide | Red Hat Enterprise Linux 7"
