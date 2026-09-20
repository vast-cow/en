---
title: "If You Have Multiple Tailscale Machines Behind the Same NAT, Change the `tailscaled` Port"
description: "When you run multiple Tailscale nodes at the same site, you may notice that only one machine..."
pubDatetime: 2026-03-11T09:57:10.384Z
---

When you run multiple Tailscale nodes at the same site, you may notice that only one machine establishes a direct connection while the others fall back to using a relay.

In environments where several hosts sit behind the same NAT, using the same `tailscaled` UDP port on every machine can make direct connectivity less reliable. In that case, assigning a different port to each machine is a practical fix.

## The Problem

By default, `tailscaled` uses the same UDP port on every host.

That is usually fine for a single machine. But when you have multiple machines in the same location, all sharing the same public IP address, NAT port mappings can become awkward. As a result, one machine may keep a clean direct path while the others end up using a relay path instead.

This is not always catastrophic, but it is suboptimal. Relay traffic generally adds latency and makes the connection path less direct than it needs to be.

## The Fix

A simple approach is to give each machine its own `tailscaled` port.

On Linux systems managed by systemd, edit:

```bash
/etc/default/tailscaled
```

Set a different port on each host.

For example, on one machine:

```bash
PORT=41641
```

On another:

```bash
PORT=41642
```

And on a third:

```bash
PORT=41643
```

Then restart the service:

```bash
sudo systemctl restart tailscaled
```

## Why This Helps

When each machine uses a distinct UDP port, NAT mappings are less likely to collide or behave in a way that prevents stable direct paths. That improves the chances that each node can establish direct peer-to-peer connectivity instead of falling back to relay.

## Things to Keep in Mind

Make sure the chosen ports do not overlap across machines at the same site.

Also confirm that local firewall rules or upstream network policies allow the UDP ports you assign.

After restarting `tailscaled`, verify the result with your usual Tailscale checks, such as connection status or ping tests, to confirm that nodes are using direct paths where expected.

## Summary

If you have multiple Tailscale machines behind the same NAT, leaving them all on the same `tailscaled` port can lead to a situation where only one host gets a direct path and the rest use relay.

A straightforward remedy is:

1. Edit `/etc/default/tailscaled`
2. Assign a unique port per machine
3. Restart the service

```bash
sudo systemctl restart tailscaled
```

It is a small change, but in multi-node same-site deployments, it can make direct connectivity much more consistent.
