---
title: "Unlocking GNOME Keyring on a Remote Machine via X11 Forwarding"
description: "Forward remote keyring prompts over SSH by updating the D-Bus display environment, unlock through Seahorse or secret-tool, and select the GCR SSH agent."
pubDatetime: 2026-08-19T09:42:29.761Z
updatedDate: 2026-08-19T10:54:07.822Z
---

A memo on unlocking the GNOME Keyring on a Linux server connected via SSH, using Seahorse through X11 Forwarding.

Also documented are how to unlock it with `secret-tool`, and how to use the SSH agent provided by GNOME Keyring / GCR from an SSH session.

## Environment

Using SSH's X11 Forwarding, display the Seahorse and GNOME Keyring password prompts launched on the remote side on the local side.

First, establish the SSH connection.

```bash
ssh -XY server
```

After connecting, also reflect the X11 environment variables into the D-Bus / systemd user session.

```bash
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

## Unlocking the GNOME Keyring

### Using Seahorse

Launch Seahorse.

```bash
seahorse
```

Once Seahorse is displayed, select

**Passwords → Login → Unlock**

and enter the GNOME Keyring password.

This unlocks the Login keyring.

### Using `secret-tool`

Instead of launching Seahorse, you can also request an unlock from `secret-tool`.

```bash
secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null
```

If the keyring is locked, a password prompt for unlocking will appear; enter the password there.

The search results themselves are not needed, so standard output is discarded to `/dev/null`.

To display the prompt via X11 Forwarding, run the following first.

```bash
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

Therefore, if you do not need to open the GUI Seahorse, just the following steps suffice.

```bash
ssh -XY server

dbus-update-activation-environment --systemd DISPLAY XAUTHORITY

secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null
```

## Using the SSH agent

To use the SSH agent on the GNOME Keyring / GCR side, point `SSH_AUTH_SOCK` to the GCR socket.

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"
```

After setting this, verify the keys recognized by the agent.

```bash
ssh-add -l
```

A series of operations looks, for example, like this.

```bash
ssh -XY server

dbus-update-activation-environment --systemd DISPLAY XAUTHORITY

secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null

export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"

ssh-add -l
```

If using Seahorse, replace the unlock part as follows.

```bash
seahorse
# Passwords → Login → Unlock
```

If you use the GCR SSH agent every time, you may add the following to your shell's configuration file.

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"
```

However, if you are also using another `ssh-agent` or agent forwarding, note that this will overwrite the existing `SSH_AUTH_SOCK`.

## Why `dbus-update-activation-environment` is needed

When using SSH's X11 Forwarding, a `DISPLAY` like the following is set in the SSH session.

```text
localhost:10.0
```

On the other hand, GUI prompts around GNOME Keyring may be launched via D-Bus or the systemd user session.

Therefore, even if `DISPLAY` is correctly set in the SSH shell, processes launched via D-Bus may not recognize the SSH X11 display.

So, run

```bash
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

to pass the current SSH session's `DISPLAY` and `XAUTHORITY` to the activation environment as well.

## Procedure Summary

When using Seahorse.

```bash
ssh -XY server

dbus-update-activation-environment --systemd DISPLAY XAUTHORITY

seahorse
# Passwords → Login → Unlock

export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"

ssh-add -l
```

When using `secret-tool`.

```bash
ssh -XY server

dbus-update-activation-environment --systemd DISPLAY XAUTHORITY

secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null

export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"

ssh-add -l
```

## Troubleshooting Notes

Check whether the GNOME Keyring daemon is running.

```bash
pgrep -af gnome-keyring-daemon
```

If the behavior is off, restart the user service.

```bash
systemctl --user stop gnome-keyring-daemon.service
systemctl --user start gnome-keyring-daemon.service
```

To watch the logs in real time, use the following.

```bash
journalctl --user-unit gnome-keyring-daemon -fe
```

Running this log in another terminal while operating Seahorse or `secret-tool` makes it easier to see what is happening on the daemon side.

For checking the SSH agent side, the following also works.

```bash
echo "$SSH_AUTH_SOCK"
ls -l "$XDG_RUNTIME_DIR/gcr/ssh"
ssh-add -l
```

The expected socket is:

```text
$XDG_RUNTIME_DIR/gcr/ssh
```

## Supplementary Notes

`ssh -X` is regular X11 Forwarding, while `ssh -Y` is trusted X11 Forwarding.

In this case, operation was confirmed with

```bash
ssh -XY server
```

Trusted X11 Forwarding grants remote applications broader permissions than regular `-X`, so it is assumed to be used only with trusted servers.

Also,

```bash
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

updates the activation environment of that user's D-Bus / systemd user session.

In environments where the same user is also using a local GUI session concurrently, be aware that this may affect where GUI applications are displayed.

Similarly,

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"
```

switches the SSH agent used from the current shell to the GCR side.

If OpenSSH's `ssh-agent` or SSH agent forwarding is already in use, the GCR-side agent will be used instead of that socket.

## Conclusion

To unlock the GNOME Keyring on a remote machine over SSH, first set up

```bash
ssh -XY server
dbus-update-activation-environment --systemd DISPLAY XAUTHORITY
```

then either run

**Passwords → Login → Unlock**

from Seahorse, or request an unlock with

```bash
secret-tool search --unlock --all xdg:schema org.freedesktop.Secret.Generic >/dev/null
```

If you also want to use the GCR SSH agent, set

```bash
export SSH_AUTH_SOCK="$XDG_RUNTIME_DIR/gcr/ssh"
ssh-add -l
```

If problems occur around the GNOME Keyring daemon, restarting it with `systemctl --user` and checking logs with `journalctl` proved effective.
