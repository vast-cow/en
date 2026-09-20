---
title: "How to Change the Hostname in WSL"
description: "Set a persistent hostname in /etc/wsl.conf and restart the distribution so WSL applies the network configuration."
pubDatetime: 2026-06-03T06:02:41.532Z
---

In WSL, you can change the hostname used within the Linux environment.

Setting the hostname to an easy-to-understand name is useful when switching between multiple WSL distributions or identifying the environment in the terminal.

## Edit the Configuration File

The WSL hostname is configured in `/etc/wsl.conf`.

Add a `[network]` section as shown below, and specify the name you want to use for `hostname`.

```ini
[network]
hostname = myhostname
```

Here, `myhostname` is specified as an example.

In practice, replace it with the hostname you want to use.

## Restart WSL

To apply the setting, restart WSL.

To terminate only a specific distribution, run the following command in the Windows-side terminal.

```powershell
wsl -t mydistribution
```

Replace `mydistribution` with the name of the target WSL distribution.

To terminate all WSL environments, use the following command.

```powershell
wsl --shutdown
```

After that, restart WSL, and the new hostname will be applied.

## Notes

In WSL, even if you change the hostname using the same methods as in a regular Linux environment, the setting may revert.

For example, the following methods may not result in a persistent change.

```bash
hostname
hostnamectl
/etc/hostname
/etc/hosts
```

To fix the hostname in WSL, using the method of setting it in `/etc/wsl.conf` is the reliable approach.
