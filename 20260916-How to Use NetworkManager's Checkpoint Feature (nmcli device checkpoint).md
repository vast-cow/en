---
title: "How to Use NetworkManager's Checkpoint Feature (nmcli device checkpoint)"
description: "nmcli device checkpoint is a feature that allows you to create a \"restore point\" for safely testing..."
pubDatetime: 2026-09-16T04:15:12.719Z
---

`nmcli device checkpoint` is a feature that allows you to create a "restore point" for safely testing changes to your NetworkManager configuration.

It's very useful when making network configuration changes on a remote server (e.g., via SSH connection) because if a configuration error causes the connection to be lost, it will automatically revert to the previous state.

---

# Basic Mechanism

Normally, when you change a network configuration:

```plaintext
Current configuration
      │
      ▼
Apply new configuration
      │
      ├── Success → Use it
      │
      └── Failure → Connection lost
```

With Checkpoint, the flow is:

```plaintext
Create Checkpoint
      │
      ▼
Apply new configuration
      │
      ├── Success
      │      │
      │      ▼
      │  Discard Checkpoint
      │
      └── Failure
             │
             ▼
      Automatically roll back after timeout
```

---

# Basic Syntax

```bash
nmcli device checkpoint create [DEVICE...] --timeout seconds
```

Example:

```bash
nmcli device checkpoint create eth0 --timeout 60
```

This means:

* Save the state of eth0
* Revert to the previous state if not confirmed within 60 seconds

---

# Practical Example

For example, if you want to change the IP address:

```bash
nmcli device checkpoint create eth0 --timeout 60
```

A checkpoint will be created.

Then, execute:

```bash
nmcli con modify eth0 \
    ipv4.addresses 192.168.1.50/24 \
    ipv4.gateway 192.168.1.1 \
    ipv4.method manual

nmcli con up eth0
```

Even if the SSH connection is lost,

```plaintext
After 60 seconds
```

NetworkManager will automatically revert to:

```plaintext
Original IP
Original Gateway
Original Route
```

---

# Confirm and Commit on Success

If the changes are successful and you can communicate, execute:

```bash
nmcli device checkpoint destroy <checkpoint-id>
```

This will:

```plaintext
Delete the checkpoint
= Do not roll back
```

---

# Manually Roll Back

If you determine that the changes failed, execute:

```bash
nmcli device checkpoint rollback <checkpoint-id>
```

to immediately revert to the previous state.

---

# Checkpoint List

You can check the current checkpoints with:

```bash
nmcli device checkpoint show
```

Example:

```plaintext
ID   CREATED              TIMEOUT
3    2025-01-01 10:00     60
```

---

# Multiple Devices are Possible

For example:

```bash
nmcli device checkpoint create eth0 bond0 br0 --timeout 120
```

This will:

```plaintext
eth0
bond0
br0
```

Save all of them together.

This is often used with bridge and bonding configurations.

---

# Meaning of `--timeout`

For example:

```bash
--timeout 30
```

means:

```plaintext
Create Checkpoint
        │
0 seconds │
10 seconds │ Apply changes
20 seconds │ Verify via SSH
30 seconds │ Automatically revert if destroy is not executed
```

---

# Typical Example in Remote Operation

When trying to change the IP address via SSH, the following procedure is safe:

```bash
nmcli device checkpoint create eth0 --timeout 120

nmcli con modify ...

nmcli con up ...

# Verify that the SSH connection is still active

nmcli device checkpoint destroy <id>
```

If the connection is lost, it will automatically revert after 120 seconds, eliminating the need for on-site recovery work.

---

# Notes

* It only applies to devices managed by NetworkManager.
* It can only roll back network state changes made by NetworkManager. Changes made outside of NetworkManager (e.g., settings added directly with the `ip` command) are not affected.
* Automatic rollback after the timeout requires the NetworkManager service to continue running.

---

## Summary

`nmcli device checkpoint` is a useful feature that can be thought of as a "safety net" for network configuration changes.

* `create`: Saves the current network state
* `show`: Displays the checkpoint list
* `destroy`: Confirms the changes (does not roll back)
* `rollback`: Immediately reverts to the saved state
* `--timeout`: Automatically rolls back if `destroy` is not executed within the specified time

It is very useful as an insurance policy against connection loss due to configuration errors when changing IP addresses, routing, bridges, and VLANs via SSH.
