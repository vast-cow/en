---
title: "Block Outbound Traffic for a Specific Linux User with iptables (While Keeping Listening Ports Working)"
description: "Use owner and connection-state rules for IPv4 and IPv6 to restrict a user's new outbound connections while permitting loopback traffic and replies from listening services."
pubDatetime: 2026-01-19T10:18:34.994Z
---

In some environments you may want to prevent a particular local user from initiating outbound network connections, while still allowing that user to run services that listen on ports and respond to inbound connections. This can be achieved with `iptables` by blocking only **NEW** outbound connections from that user, while allowing **ESTABLISHED/RELATED** traffic to continue.

This article shows a clean, minimal ruleset to accomplish that goal.

## Goal

For **userA**:

* Block **new outbound connections** (e.g., `curl`, package downloads, external API calls).
* Keep **port listening** and **inbound service responses** working (e.g., a daemon binding to `0.0.0.0:8080` and responding to remote clients).

The key is to add a rule in the **OUTPUT** chain that matches the user and drops/rejects only traffic in **conntrack state NEW**.



## Prerequisites

* Root privileges (`sudo`)
* `conntrack` support (available on most modern Linux systems)
* The user’s UID (recommended for accuracy)



## 1) Identify the UID of the User

Get the numeric UID for `userA`:

```bash
id -u userA
```

In the commands below, replace `<UID_A>` with the returned number.



## 2) Apply iptables Rules (IPv4)

### Allow established traffic first (important)

This ensures return traffic for existing inbound connections (and other established flows) is not disrupted.

```bash
sudo iptables -I OUTPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

### Allow loopback traffic (recommended)

Many local services rely on `lo` (127.0.0.1). Keeping it open avoids breaking internal IPC patterns over TCP.

```bash
sudo iptables -I OUTPUT 2 -o lo -j ACCEPT
```

### Block NEW outbound connections for the user

This is the core rule.

```bash
sudo iptables -I OUTPUT 3 -m owner --uid-owner <UID_A> -m conntrack --ctstate NEW -j REJECT
```

Notes:

* `REJECT` fails fast and is easier to debug. If you prefer silent blocking, replace `REJECT` with `DROP`.
* The `owner` match works only for locally generated packets (OUTPUT), which is exactly what you want here.



## Why This Keeps Listening Ports Working

A user process can still:

* Bind and listen on a port (listening itself is not a network transmission).
* Accept inbound connections initiated by remote clients.

When a remote host connects to your service, the responses from your server are part of an already established connection. Those response packets are classified as **ESTABLISHED**, so they are permitted by the first rule.

As a result:

* Outbound connection initiation is blocked.
* Inbound services can still function normally.



## 3) Verify the Rules

Check the active OUTPUT chain rules:

```bash
sudo iptables -S OUTPUT
sudo iptables -L OUTPUT -n -v --line-numbers
```

Testing ideas:

* As `userA`, try:

  * `curl https://example.com` (should fail)
* From another host, connect to a service listening under `userA`:

  * e.g., `nc <server-ip> 8080` (should connect and receive responses)



## 4) Don’t Forget IPv6 (Common Bypass)

If IPv6 is enabled, traffic may still escape via IPv6 unless you apply equivalent `ip6tables` rules:

```bash
sudo ip6tables -I OUTPUT 1 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo ip6tables -I OUTPUT 2 -o lo -j ACCEPT
sudo ip6tables -I OUTPUT 3 -m owner --uid-owner <UID_A> -m conntrack --ctstate NEW -j REJECT
```



## Operational Considerations and Caveats

### 1) `-m owner` applies only to OUTPUT

The `owner` module matches processes on the local host and is only meaningful for locally generated traffic. It does not apply to forwarded traffic (FORWARD), and it won’t help if you’re trying to police routing/NAT for other hosts.

### 2) Privilege escalation can bypass the restriction

If `userA` can run commands as another user (e.g., `sudo` to root), the traffic will be owned by a different UID and won’t match this rule. If that’s a concern, address it via sudo policy and system hardening.

### 3) Existing connections may remain open

Because the rule blocks only `NEW`, any already-established outbound connections may continue until they close naturally. If you need immediate cutoff, you’ll have to terminate the processes or kill existing connections (e.g., via `ss`/`lsof` workflows) in addition to adding rules.



## Conclusion

Blocking outbound traffic for a specific Linux user while keeping listening ports functional is straightforward with `iptables`:

* Allow `ESTABLISHED,RELATED`
* Allow loopback
* Reject or drop user-owned `NEW` outbound connections in OUTPUT

This approach is minimally invasive and well-suited for service accounts where you want to limit egress without breaking inbound service behavior.
