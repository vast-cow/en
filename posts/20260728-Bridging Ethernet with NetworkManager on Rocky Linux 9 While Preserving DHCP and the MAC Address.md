---
title: "Bridging Ethernet with NetworkManager on Rocky Linux 9 While Preserving DHCP and the MAC Address"
description: "Stage a NetworkManager bridge migration with the existing MAC address and DHCP, test activation through a checkpoint, and verify persistence before retiring the old profile."
pubDatetime: 2026-07-28T12:31:43.829Z
updatedDate: 2026-09-16T04:13:14.626Z
---

On Ubuntu, it is common to use Netplan to attach an Ethernet interface to a Linux bridge. On Rocky Linux 9, however, networking is typically managed by **NetworkManager**.

Instead of Netplan, Rocky Linux uses tools such as `nmcli` to create a bridge and attach the physical network interface to it.

This article explains how to create the equivalent of an Ubuntu Netplan bridge configuration on Rocky Linux 9 while preserving DHCP, preserving the original Ethernet MAC address, and using a NetworkManager checkpoint to protect against losing remote connectivity.

## Network Topology

For example, if the wired network interface is:

```text
eth0
```

the final configuration will look like this:

```text
          DHCP Server
               │
        Ethernet Switch
               │
             eth0
               │
         Linux Bridge
             br0
               │
      IP Address (DHCP)
               │
         hostapd (AP)
               │
        Wi-Fi Clients
```

With this configuration:

* The physical NIC becomes a port of the bridge.
* The bridge (`br0`) owns the IP address.
* DHCP runs on `br0`, not `eth0`.
* The bridge uses the MAC address previously used by `eth0`.
* `hostapd` can attach the wireless interface to the bridge.
* Wi-Fi clients join the same Layer 2 network as the wired LAN.

## Identify the Existing Ethernet Connection

Before changing anything, determine which NetworkManager connection profile is currently active on `eth0`.

```bash
nmcli device status
```

You can also retrieve the active profile directly:

```bash
nmcli -g GENERAL.CONNECTION device show eth0
```

For example, the result might be:

```text
Wired connection 1
```

Save this name for later. In the commands below, it is represented as:

```text
<existing-profile-name>
```

Do **not** delete this profile yet. It provides the known-good configuration that NetworkManager can restore if the bridge migration fails.

## Record the Existing MAC Address

Determine the current permanent MAC address of `eth0`:

```bash
cat /sys/class/net/eth0/address
```

For example:

```text
00:11:22:33:44:55
```

This value will be assigned to the bridge.

You can optionally store it in a shell variable:

```bash
ETH_MAC=$(cat /sys/class/net/eth0/address)
```

## Create the Bridge Without Activating It

Create the bridge profile:

```bash
sudo nmcli connection add \
    type bridge \
    ifname br0 \
    con-name br0
```

Because the configuration is being prepared remotely, disable automatic activation while it is being staged:

```bash
sudo nmcli connection modify br0 \
    connection.autoconnect no
```

This prevents NetworkManager from unexpectedly activating an incomplete bridge configuration.

## Preserve the Bridge MAC Address

A newly created bridge may otherwise use a different MAC address.

Configure the bridge to use the same MAC address as the original Ethernet interface:

```bash
sudo nmcli connection modify br0 \
    bridge.mac-address "$ETH_MAC"
```

Or specify the address directly:

```bash
sudo nmcli connection modify br0 \
    bridge.mac-address 00:11:22:33:44:55
```

Preserving the MAC address helps maintain the host's existing network identity.

## Configure DHCP on the Bridge

The bridge—not the physical NIC—should obtain the IP address.

Configure IPv4 DHCP and disable IPv6 if that matches the intended network configuration:

```bash
sudo nmcli connection modify br0 \
    ipv4.method auto \
    ipv6.method disabled
```

If IPv6 is required on the network, configure it appropriately instead of disabling it.

## Create the Ethernet Bridge Port

Create a separate NetworkManager profile that attaches `eth0` to `br0`:

```bash
sudo nmcli connection add \
    type bridge-slave \
    ifname eth0 \
    master br0 \
    con-name br0-port-eth0
```

Disable autoconnect on this profile while the new configuration is being staged:

```bash
sudo nmcli connection modify br0-port-eth0 \
    connection.autoconnect no
```

At this point, the new bridge configuration exists on disk but should not yet have replaced the currently active Ethernet connection.

Verify the profiles:

```bash
nmcli connection show
```

You should see entries similar to:

```text
NAME                 TYPE      DEVICE
Wired connection 1   ethernet  eth0
br0                  bridge    --
br0-port-eth0        ethernet  --
```

The exact output depends on the NetworkManager version and the existing configuration.

## Test the Migration with a NetworkManager Checkpoint

The disruptive part of the migration is activating `eth0` as a bridge port.

When performing this operation over SSH, use a NetworkManager checkpoint:

```bash
sudo nmcli device checkpoint --timeout 120 -- \
    nmcli connection up br0-port-eth0
```

Activating a bridge port causes NetworkManager to activate its bridge controller as required.

Because `eth0` can only use one NetworkManager connection profile at a time, activating `br0-port-eth0` replaces the currently active standalone Ethernet profile on `eth0`.

The SSH connection may briefly pause while this happens.

NetworkManager will then ask whether the change should be kept.

If the new configuration works and the SSH connection remains available, answer:

```text
Yes
```

If the network configuration breaks and the SSH connection is lost, you will not be able to confirm the checkpoint. After the timeout expires, NetworkManager attempts to restore the network state that existed before the command was executed.

The default checkpoint timeout is short, so specifying a longer timeout such as 120 seconds is useful when DHCP needs time to complete.

## Verify the Bridge Before Confirming

Before answering `Yes`, verify the configuration from another SSH session if possible.

Check the bridge address:

```bash
ip addr show br0
```

Check the bridge port:

```bash
bridge link
```

Check NetworkManager:

```bash
nmcli device status
```

Check the default route:

```bash
ip route
```

The expected state is:

* `eth0` is a port of `br0`.
* `br0` owns the IPv4 address.
* DHCP has assigned an address to `br0`.
* The default route uses `br0`.
* The bridge has the original Ethernet MAC address.
* The remote SSH connection still works.

You can check the bridge MAC address with:

```bash
ip link show br0
```

and compare it with:

```bash
cat /sys/class/net/eth0/address
```

Once these checks succeed, answer `Yes` to the checkpoint prompt.

## Make the Working Configuration Persistent

The test profiles were deliberately configured with autoconnect disabled.

After the checkpoint has succeeded and connectivity has been confirmed, enable autoconnect on the bridge and its Ethernet port:

```bash
sudo nmcli connection modify br0 \
    connection.autoconnect yes

sudo nmcli connection modify br0-port-eth0 \
    connection.autoconnect yes
```

Then disable automatic activation of the old standalone Ethernet profile:

```bash
sudo nmcli connection modify "<existing-profile-name>" \
    connection.autoconnect no
```

Keeping the old profile temporarily is safer than immediately deleting it.

Verify the resulting settings:

```bash
nmcli -f NAME,TYPE,AUTOCONNECT connection show
```

You should now have:

```text
br0                  bridge    yes
br0-port-eth0        ethernet  yes
<existing-profile>   ethernet  no
```

## Reboot Test

A successful live migration does not by itself prove that the system will boot with the intended configuration.

If remote access is important, ensure an out-of-band console is available before performing the first reboot.

Then reboot:

```bash
sudo reboot
```

After the system returns, verify:

```bash
nmcli device status
```

```bash
ip addr show br0
```

```bash
bridge link
```

```bash
ip route
```

The expected configuration is:

* `br0` is active.
* `eth0` is attached to `br0`.
* `br0` has the DHCP address.
* The default route uses `br0`.
* The old standalone Ethernet profile remains inactive.

## Remove the Old Ethernet Profile

Only after the bridge has survived a reboot and has been fully verified should the old standalone Ethernet profile be removed, if desired.

```bash
sudo nmcli connection delete "<existing-profile-name>"
```

Deleting it is optional. Leaving it present with:

```text
connection.autoconnect no
```

can be useful as a recovery configuration.

## Why Preserve the MAC Address?

When a bridge is created, it may use a different MAC address from the original Ethernet interface.

If the MAC address changes:

* The DHCP server may recognize the system as a different device.
* A new DHCP lease may be assigned.
* DHCP reservations tied to the old MAC address may stop working.
* MAC address-based access control may no longer match.
* Neighboring systems may need to refresh their ARP or neighbor cache entries.

To minimize changes to the existing network, configure the bridge to use the MAC address previously used by the physical NIC:

```bash
bridge.mac-address 00:11:22:33:44:55
```

The Layer 3 identity then moves from `eth0` to `br0` while retaining the same Layer 2 address.

## Using the Bridge with hostapd

After creating and validating the bridge, specify it in `hostapd.conf`:

```ini
interface=wlp2s0
bridge=br0
```

This allows:

* Wired LAN devices
* Wi-Fi clients
* The upstream DHCP server

to operate on the same Layer 2 network.

Wi-Fi clients can therefore obtain IP addresses directly from the existing DHCP server rather than requiring a separate routed or NAT network.

## The Configuration Persists Across Reboots

A bridge created using `nmcli` is not merely a temporary kernel configuration.

NetworkManager stores persistent connection profiles under:

```text
/etc/NetworkManager/system-connections/
```

Rocky Linux 9 uses NetworkManager keyfile profiles for newly created connections.

For example:

```bash
sudo nmcli connection add \
    type bridge \
    ifname br0 \
    con-name br0
```

normally results in a NetworkManager connection profile being stored under:

```text
/etc/NetworkManager/system-connections/
```

The bridge-port profile is stored persistently as well.

By contrast, creating a bridge directly with:

```bash
ip link add br0 type bridge
```

only creates a kernel network interface. Unless another configuration mechanism recreates it during boot, that interface disappears after reboot.

## Precautions When Reconfiguring Over SSH

Changing the interface that carries an SSH session is inherently risky.

NetworkManager provides the `device checkpoint` command specifically to make disruptive network changes safer.

The general form is:

```bash
nmcli device checkpoint [--timeout SECONDS] [DEVICE...] -- COMMAND
```

NetworkManager takes a checkpoint, runs the specified command, and asks whether the resulting network state should be kept.

If confirmation is not received before the timeout, NetworkManager attempts to restore the previous state.

For this bridge migration:

```bash
sudo nmcli device checkpoint --timeout 120 -- \
    nmcli connection up br0-port-eth0
```

provides behavior similar in purpose to Ubuntu's:

```bash
sudo netplan try
```

There are some important precautions:

* Do not delete the original Ethernet profile before testing the bridge.
* Stage new profiles with `connection.autoconnect no`.
* Use a sufficiently long checkpoint timeout for DHCP to complete.
* Confirm that `br0` has an address, route, and working connectivity before accepting the checkpoint.
* Only enable autoconnect on the new profiles after the live test succeeds.
* Disable rather than immediately delete the original Ethernet profile.
* Test the configuration across a reboot before removing the old profile.
* Use an out-of-band console such as IPMI, iDRAC, iLO, or a hypervisor console whenever possible.

`tmux` or `screen` can still be useful for other administrative work, but they do not protect against a broken network configuration. The NetworkManager checkpoint is what provides the automatic network-state rollback.

## Rocky Linux 9 Version Differences

Recent RHEL 9 and Rocky Linux 9 releases use the terminology **controller** and **port** for relationships such as bridges.

For example, newer syntax can use:

```bash
sudo nmcli connection add \
    type ethernet \
    ifname eth0 \
    con-name br0-port-eth0 \
    port-type bridge \
    controller br0
```

Older Rocky Linux 9 releases use the equivalent `bridge-slave` / `master` terminology:

```bash
sudo nmcli connection add \
    type bridge-slave \
    ifname eth0 \
    master br0 \
    con-name br0-port-eth0
```

The latter form is useful when instructions need to cover older Rocky Linux 9 installations as well.

## Netplan and NetworkManager Comparison

| Netplan         | NetworkManager (`nmcli`)             |
| --------------- | ------------------------------------ |
| `bridges:`      | `type bridge`                        |
| `interfaces:`   | bridge port / `bridge-slave`         |
| `dhcp4: true`   | `ipv4.method auto`                   |
| `dhcp6: false`  | `ipv6.method disabled`               |
| `macaddress:`   | `bridge.mac-address`                 |
| `netplan apply` | `nmcli connection up ...`            |
| `netplan try`   | `nmcli device checkpoint -- COMMAND` |

The exact implementations differ, but both provide a way to test potentially disruptive network changes with rollback protection.

## Summary

On Rocky Linux 9, NetworkManager's `nmcli` can create the same type of Ethernet bridge commonly configured through Netplan on Ubuntu.

The important points are:

* Assign the IP configuration to the bridge rather than the physical NIC.
* Configure the bridge to use the MAC address previously used by the Ethernet NIC when preserving network identity is important.
* Create a separate bridge-port profile for the Ethernet interface.
* Do not delete the known-good Ethernet profile before testing the bridge.
* Stage the replacement profiles with autoconnect disabled.
* Use `nmcli device checkpoint` when performing the disruptive activation over SSH.
* Verify DHCP, routing, the bridge MAC address, and SSH connectivity before accepting the checkpoint.
* Enable autoconnect only after the bridge has been successfully tested.
* Disable the old Ethernet profile first and delete it only after the new configuration has survived a reboot.
* Point `hostapd` to `br0` so Wi-Fi clients can join the same Layer 2 network as the wired LAN.
* NetworkManager profiles created with `nmcli` persist across reboots.

With this setup, wired LAN devices and Wi-Fi clients share the same Layer 2 network, while NetworkManager's checkpoint mechanism substantially reduces the risk of permanently losing remote connectivity during the migration.
