---
title: "Safely Backing Up and Restoring iptables and ip6tables Rules on Linux"
description: "When experimenting with firewall rules, it is essential to have a reliable way to revert to a..."
pubDatetime: 2026-01-19T10:14:56.429Z
---

When experimenting with firewall rules, it is essential to have a reliable way to revert to a known-good configuration. This article explains how to **save the current iptables / ip6tables state**, **apply temporary changes**, and **restore the original rules** if needed.

All commands shown below must be executed with **root privileges**.



## 1. Backing Up the Current Firewall Rules

The most reliable way to back up firewall rules is to use `iptables-save` and `ip6tables-save`. These commands dump the entire ruleset in a format suitable for restoration.

### IPv4

```bash
iptables-save > /root/iptables.bak
```

### IPv6

```bash
ip6tables-save > /root/ip6tables.bak
```

Optionally, you can include a timestamp for traceability:

```bash
iptables-save  > /root/iptables.bak.$(date +%F_%H%M%S)
ip6tables-save > /root/ip6tables.bak.$(date +%F_%H%M%S)
```



## 2. Applying Temporary Changes

Once the backup is complete, you can safely experiment with firewall rules.

Example: temporarily allow SSH traffic (TCP port 22) at the top of the INPUT chain.

```bash
iptables  -I INPUT 1 -p tcp --dport 22 -j ACCEPT
ip6tables -I INPUT 1 -p tcp --dport 22 -j ACCEPT
```

Verify the current ruleset:

```bash
iptables  -S
ip6tables -S
```

**Tip:**
When working over SSH, always start with non-disruptive changes (such as adding `ACCEPT` or `LOG` rules) to avoid locking yourself out.



## 3. Restoring the Original Rules

If the changes do not behave as expected, you can restore the original configuration instantly using `iptables-restore` and `ip6tables-restore`.

```bash
iptables-restore  < /root/iptables.bak
ip6tables-restore < /root/ip6tables.bak
```

Confirm that the rules have been restored:

```bash
iptables-save
ip6tables-save
```



## 4. Important Notes and Caveats

* **iptables-nft backend**
  On modern distributions, `iptables` may be backed by nftables (`iptables-nft`). The save/restore commands still work correctly and will update the nftables ruleset internally.

* **Firewall management services**
  If services such as `firewalld` or `ufw` are running, they may automatically overwrite manual changes.
  Check their status before testing:

  ```bash
  systemctl is-active firewalld
  systemctl is-active ufw
  ```

* **Fail-safe recovery**
  For remote systems, consider scheduling an automatic rollback (e.g., via `at` or `systemd-run`) before applying risky rules. This ensures recovery even if network access is lost.



## Conclusion

By using `iptables-save` and `iptables-restore` (and their IPv6 equivalents), you can safely experiment with firewall rules and always return to a stable baseline. This workflow is simple, fast, and highly recommended for both testing and troubleshooting scenarios.
