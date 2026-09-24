---
title: "Running `sshd` in Rootless Mode"
description: "Create a user-owned SSH server configuration and host key, restrict access to public-key authentication for the current user, and run foreground SSH/SFTP on a high port."
pubDatetime: 2026-04-16T05:36:51.738Z
---

This article explains how to run `sshd` as a normal user without root privileges. The setup uses a custom configuration directory under the user’s home directory and listens on a non-privileged port.

## Overview

Normally, `sshd` is started by the system as root and listens on port `22`. In some environments, however, it is useful to run an SSH server without root access. A rootless setup can be used for testing, development, or temporary remote access in user space.

The example here creates a private SSH server environment in `~/.sshd`, generates a host key if needed, writes a minimal `sshd_config`, validates the configuration, and then starts the daemon in the foreground.

## Directory and Variable Setup

The `setup.sh` script begins by defining a few variables:

```bash
BASE="$HOME/.sshd"
PORT=2222
SSHD="$(command -v sshd)"
```

`BASE` is the working directory for the SSH server files. `PORT` is set to `2222`, which is a non-privileged port and can be used without root. `SSHD` stores the path to the `sshd` executable.

The script then creates the required directories:

```bash
install -d -m 700 "$BASE" "$HOME/.ssh"
```

This ensures that both `~/.sshd` and `~/.ssh` exist with secure permissions.

## Generating the Host Key

An SSH server needs its own host key. The script checks whether an Ed25519 host key already exists:

```bash
if [ ! -f "$BASE/ssh_host_ed25519_key" ]; then
  ssh-keygen -q -t ed25519 -N '' -f "$BASE/ssh_host_ed25519_key"
fi
chmod 600 "$BASE/ssh_host_ed25519_key"
```

If the key is missing, it generates one. The permission is then restricted to the owner only. This is important because SSH rejects private keys that are too broadly accessible.

## Creating the `sshd` Configuration

The script writes a custom `sshd_config` file into the user-owned directory.

```bash
cat > "$BASE/sshd_config" <<EOF
Port $PORT
ListenAddress 0.0.0.0

UsePAM no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes

HostKey $BASE/ssh_host_ed25519_key
PidFile none

PermitRootLogin no
PrintMotd no
PrintLastLog no
X11Forwarding no
AllowUsers $USER

Subsystem sftp internal-sftp
EOF
```

### Key Settings

#### Port and Listen Address

* `Port $PORT` tells the server to listen on port `2222`.
* `ListenAddress 0.0.0.0` allows connections on all network interfaces.

#### Authentication

* `UsePAM no` disables PAM because PAM generally requires a system-level setup.
* `PasswordAuthentication no` disables password login.
* `KbdInteractiveAuthentication no` disables keyboard-interactive authentication.
* `PubkeyAuthentication yes` enables public key login.

This makes the server simpler and safer for rootless use by relying only on SSH keys.

#### Host Key and PID File

* `HostKey $BASE/ssh_host_ed25519_key` points to the host key generated earlier.
* `PidFile none` avoids writing a PID file, which is useful in lightweight user-space execution.

#### Restrictions

* `PermitRootLogin no` disallows root login.
* `PrintMotd no` and `PrintLastLog no` suppress extra login messages.
* `X11Forwarding no` disables X11 forwarding.
* `AllowUsers $USER` limits access to the current user only.

#### SFTP Support

* `Subsystem sftp internal-sftp` enables SFTP using the built-in internal subsystem.

## Validating the Configuration

At the end of `setup.sh`, the script verifies the configuration:

```bash
"$SSHD" -t -f "$BASE/sshd_config"
```

This is a useful step because it checks the syntax before starting the server.

## Starting the SSH Server

The `start.sh` script is very small:

```bash
#!/bin/bash
BASE="$HOME/.sshd"
SSHD="$(command -v sshd)"
exec "$SSHD" -D -e -f "$BASE/sshd_config"
```

### What This Does

* `-D` keeps `sshd` in the foreground.
* `-e` sends log output to standard error.
* `-f "$BASE/sshd_config"` tells `sshd` to use the custom configuration file.

Using `exec` replaces the shell process with `sshd`, which is a clean way to launch the server.

## How to Use It

A typical flow is:

1. Run `setup.sh` once to create the configuration and host key.
2. Make sure your public key is present in `~/.ssh/authorized_keys`.
3. Run `start.sh` to launch the SSH server.
4. Connect to it on port `2222`.

For example:

```bash
ssh -p 2222 user@host
```

## Benefits of This Approach

This rootless SSH setup has several advantages:

* It does not require system-wide configuration.
* It avoids privileged ports.
* It works well for personal environments and temporary setups.
* It keeps all SSH server files under the user’s home directory.

## Conclusion

This example provides a compact way to run `sshd` without root privileges. By using a private configuration directory, a non-privileged port, and public key authentication only, it creates a simple and practical SSH server for user-space operation.

## Scripts 

setup.sh 
```bash
#!/bin/bash 
BASE="$HOME/.sshd" 
PORT=2222 
SSHD="$(command -v sshd)" 
 
install -d -m 700 "$BASE" "$HOME/.ssh" 
 
# server host key 
if [ ! -f "$BASE/ssh_host_ed25519_key" ]; then 
  ssh-keygen -q -t ed25519 -N '' -f "$BASE/ssh_host_ed25519_key" 
fi 
chmod 600 "$BASE/ssh_host_ed25519_key" 
 
cat > "$BASE/sshd_config" <<EOF 
Port $PORT 
ListenAddress 0.0.0.0 
 
UsePAM no 
PasswordAuthentication no 
KbdInteractiveAuthentication no 
PubkeyAuthentication yes 
 
HostKey $BASE/ssh_host_ed25519_key 
PidFile none 
 
PermitRootLogin no 
PrintMotd no 
PrintLastLog no 
X11Forwarding no 
AllowUsers $USER 
 
Subsystem sftp internal-sftp 
EOF 
 
"$SSHD" -t -f "$BASE/sshd_config" 
``` 
 
start.sh 
```bash
#!/bin/bash 
BASE="$HOME/.sshd" 
SSHD="$(command -v sshd)" 
exec "$SSHD" -D -e -f "$BASE/sshd_config" 
```
