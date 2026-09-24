---
title: "Capturing DHCP-Assigned Network Parameters Using a Temporary macvlan in a Linux Network Namespace"
description: "Probe DHCP configuration in an isolated network namespace, export the resulting MAC, IP addresses, and gateways to a shell file, and remove the temporary interfaces afterward."
pubDatetime: 2026-01-19T11:08:36.658Z
---

This article explains a simple workflow for safely obtaining network configuration information—such as a MAC address, newly assigned IPv4/IPv6 addresses, and default gateways—by creating a temporary **macvlan** interface, moving it into a **Linux network namespace**, running **DHCP**, recording results into a reusable `env.sh` file, and then cleaning up all temporary objects.

## Overview

The goal is to:

1. Create a **macvlan** interface on top of a physical interface (e.g., `eth0`).
2. Move that macvlan interface into a dedicated network namespace (e.g., `netns1`).
3. Run `dhclient` inside the namespace to obtain addresses:

   * IPv4 via DHCP (`dhclient`)
   * IPv6 via DHCPv6 if available (`dhclient -6`), while still allowing IPv6 Router Advertisements (SLAAC) to populate addresses where DHCPv6 is not used.
4. Compare the interface’s address state **before and after** DHCP, and record only the **newly added** IPv4/IPv6 address (the first added address).
5. Collect the **final default route gateway** (“via”) for IPv4 and IPv6 (without before/after comparison).
6. Save values into `env.sh` in a format that later scripts can source.
7. Delete the temporary macvlan and namespace to leave the system unchanged.

## Why Use a Network Namespace?

A network namespace provides an isolated networking context. Running DHCP inside the namespace prevents your host’s primary networking from being modified and makes it easy to discard all state afterward. This is particularly helpful when you only need to “probe” a network and capture the resulting configuration for later use.

## Why Use `ip -j` and `jq`?

The `ip` command supports JSON output (`-j`), which is much easier and safer to parse than human-readable output. With `jq`, you can reliably extract:

* The interface MAC address
* IPv4/IPv6 addresses in CIDR form
* Default route gateways (“via”) for IPv4 and IPv6

This approach avoids brittle text parsing and is more robust across distributions and `iproute2` versions.

## What Gets Written to `env.sh`?

The script writes shell-exportable variables, including:

* `MACADDR`: the macvlan interface’s MAC address
* `IPV4_ADDR_ADDED_1`: the first IPv4 address that appeared after DHCP (if any)
* `IPV6_ADDR_ADDED_1`: the first IPv6 address that appeared after DHCP/RA (if any)
* `DEFAULT_VIA4`: the IPv4 default gateway from the final routing table (if any)
* `DEFAULT_VIA6`: the IPv6 default gateway from the final routing table (if any)

Because these are standard `export` assignments, later scripts can simply do:

```bash
source ./env.sh
```

and immediately reuse the captured parameters.

## Cleanup and Safety

A key design point is that the macvlan interface and namespace are temporary. The script uses a cleanup routine (typically via `trap`) to ensure the following objects are removed even if an error occurs:

* The macvlan interface inside the namespace
* The namespace itself

This ensures the host returns to its original state after the run.

## Practical Notes

* IPv6 behavior varies by network:

  * Some networks provide IPv6 via DHCPv6, some via SLAAC/RA, and some both.
  * A short wait after running DHCP helps capture addresses that arrive via Router Advertisements.
* If multiple default routes exist, the script records the first gateway returned by `ip` (unless enhanced to pick the lowest metric).

## Conclusion

By combining macvlan, network namespaces, DHCP, and JSON parsing with `jq`, you can safely obtain network configuration data without permanently altering the host’s networking. Saving results in `env.sh` provides a clean handoff to other scripts and automation, while the final cleanup step keeps the system tidy and repeatable.
