---
title: "Bridging Ethernet with Netplan While Preserving DHCP and the MAC Address"
description: "When using a Linux machine as a Wi-Fi access point with hostapd, you may want Wi-Fi clients to join..."
pubDatetime: 2026-07-28T12:23:28.743Z
---

When using a Linux machine as a Wi-Fi access point with **hostapd**, you may want Wi-Fi clients to join the same Layer 2 (L2) network as your existing wired LAN.

A common approach is to add the Ethernet interface to a Linux bridge and assign the IP address to the bridge instead of the physical interface.

## Example Netplan Configuration

For example, if your Ethernet interface is `eth0`, configure it as follows:

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    eth0:
      dhcp4: false
      dhcp6: false

  bridges:
    br0:
      interfaces:
        - eth0
      dhcp4: true
      dhcp6: false
      macaddress: xx:xx:xx:xx:xx:xx
```

This configuration does the following:

* Makes `eth0` a member port of the bridge.
* Allows `br0` to obtain its IP address via DHCP.
* Sets the MAC address of `br0` to match the original Ethernet interface.

### Why Fix the MAC Address?

When a bridge is created, some environments assign a new MAC address to `br0`.

If the MAC address changes, it may result in:

* The DHCP server recognizing it as a new client.
* A different DHCP lease being assigned.
* MAC address–based access control rules no longer matching.
* ARP caches being refreshed.

To minimize the impact on the existing network, explicitly configure the bridge to use the original Ethernet interface's MAC address:

```yaml
macaddress: xx:xx:xx:xx:xx:xx
```

## Safely Testing the Configuration

When modifying network settings over a remote SSH session, an incorrect configuration can cause you to lose connectivity.

Instead of immediately running `netplan apply`, it is safer to use `netplan try`:

```bash
sudo netplan try --timeout 30
```

This command temporarily applies the configuration. If you confirm that everything is working within 30 seconds, the changes become permanent.

If the connection is lost or you do not confirm the changes, Netplan automatically restores the previous configuration after the timeout expires.

This makes it much safer to test network configuration changes over SSH.

## Using the Bridge with hostapd

After creating the bridge, configure **hostapd** to use it:

```ini
interface=wlp2s0
bridge=br0
```

With this configuration, Wi-Fi clients connect to the wired LAN through `br0`, placing them on the same Layer 2 network.

As a result:

* Wired devices
* Wi-Fi devices
* The DHCP server

all operate on the same network segment, allowing Wi-Fi clients to obtain IP addresses directly from the existing DHCP server.

## Summary

When using **hostapd** on Linux, a bridged network configuration is a simple and maintainable solution.

To minimize disruption to your existing network, keep these three points in mind:

* Assign the IP address to the bridge rather than the physical Ethernet interface.
* Configure the bridge to use the same MAC address as the original Ethernet interface.
* Use `netplan try` during remote administration to safely test configuration changes before committing them.
