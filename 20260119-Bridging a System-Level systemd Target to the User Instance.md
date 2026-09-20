---
title: "Bridging a System-Level systemd Target to the User Instance"
description: "When working with systemd, it is common to rely on network-online.target to ensure services start..."
pubDatetime: 2026-01-19T14:49:24.325Z
---

When working with systemd, it is common to rely on `network-online.target` to ensure services start only after the network is fully up. However, this target exists at the *system* level, while many modern workflows run long-lived services in the *user* systemd instance (for example, user-managed daemons, development tooling, or per-user agents). This setup provides a clean bridge: when the system reaches “network online,” it triggers a corresponding marker target inside a specific user’s systemd instance—automatically and reliably at boot.

## Overview of the Approach

The solution consists of three parts:

1. A small installation script (`install.sh`) that sets up required system and user units.
2. A user-level marker unit (`network-online.target`) that exists only to represent “network online” inside the user instance.
3. A system-level templated service (`user-network-online@.service`) that waits for system networking to be online and then starts the user-level target for a given user ID (UID).

Together, these components ensure that user services can depend on `network-online.target` in the user instance, even though the real network readiness signal originates at the system level.

## Enabling User systemd Without an Active Login

A key challenge with user systemd instances is lifecycle: by default, they are tied to interactive logins. If the user is not logged in, the user systemd instance may not exist, which prevents user services from starting at boot.

The installation script addresses this by enabling *linger*:

* `loginctl enable-linger $USER`

With linger enabled, the user’s systemd instance can run even when no session is active. This is a prerequisite for starting user services during the boot process.

## Creating a User-Level Network-Online Marker

In the user systemd directory (`~/.config/systemd/user`), the setup installs a minimal `network-online.target` unit:

```ini
[Unit]
Description=User-level Network is Online (marker only)
Documentation=man:systemd.special(7)
```

This unit is intentionally simple: it does not implement networking logic or checks. Its purpose is to provide a synchronization point so that user services can declare dependencies like:

* `After=network-online.target`
* `Wants=network-online.target`

In other words, it acts as a “ready” flag inside the user instance.

## Bridging the System Target to the User Instance

The real bridging happens via the system-level service template `user-network-online@.service`:

```ini
[Unit]
Description=Trigger user-level network-online.target for %i
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=%i
Environment=XDG_RUNTIME_DIR=/run/user/%i
ExecStart=/usr/bin/systemctl --user start network-online.target

[Install]
WantedBy=multi-user.target
```

### How It Works

* The service runs *after* the system’s `network-online.target`, meaning it triggers only once the system considers networking ready.
* It runs as a one-shot task and executes:

  * `systemctl --user start network-online.target`
* The `XDG_RUNTIME_DIR=/run/user/%i` environment is critical. It points `systemctl --user` at the correct runtime directory for the target user instance, ensuring the command reaches the intended user systemd manager rather than failing due to missing session context.

## Ensuring It Starts Automatically at Boot

The install script wires everything together:

```sh
#!/bin/sh

loginctl enable-linger $USER

sudo cp -t /etc/systemd/system/ user-network-online@.service
sudo systemctl daemon-reload

USER_SYSTEMD_PATH="$HOME"/.config/systemd/user
mkdir -p "$USER_SYSTEMD_PATH"
cp -t "$USER_SYSTEMD_PATH" network-online.target
systemctl --user daemon-reload

sudo systemctl enable --now user-network-online@$(id -u).service
```

### Key Steps

* Installs the system unit into `/etc/systemd/system/` and reloads systemd.
* Installs the user target into `~/.config/systemd/user/` and reloads the user daemon.
* Enables and starts the templated system service for the current user’s UID:

  * `user-network-online@<UID>.service`

Because the unit is installed with `WantedBy=multi-user.target`, it will run during the normal boot sequence. As soon as the system reaches `network-online.target`, it triggers the user’s `network-online.target`, allowing user services to start in an orderly and dependency-driven way.

## Practical Outcome

With this bridge in place:

* The system decides when networking is “online.”
* The user systemd instance receives an equivalent “network online” signal.
* User services can safely depend on `network-online.target` without implementing their own network checks.
* Everything starts automatically at boot, even when the user is not logged in, thanks to linger.

This pattern is lightweight, explicit, and aligns well with systemd’s dependency model—making it a solid foundation for user-managed background services that must wait for reliable network availability.
