---
pubDatetime: 2026-01-19T07:51:00
title: "Pre-experiment: Floating IP/MAC handoff for an on-demand (Suspend + WoL) access system"
description: A pre-experiment validating whether a fixed service IP/MAC can be handed off between an always-on standby node and a suspended target machine while preserving TCP connection attempts. Using Linux network namespaces, macvlan, and controlled packet dropping, the experiment examines TCP retry behavior, ARP/FDB convergence, and the requirements for building an on-demand Suspend + Wake-on-LAN access system.
---

## Goal

I want to build a system where a target machine stays in **Suspend** while idle, and is brought up **on demand** via **Wake on LAN (WoL)** when a client needs access. From the client’s point of view, access should always be done against a stable destination—without reconfiguration—and the system should tolerate the wake-up gap as gracefully as possible.

To validate the feasibility, I ran a pre-experiment to confirm that **TCP connection attempts can be completed correctly** when a fixed **service MAC address (mac0)** and **service IP address (ip0)** are “handed off” from one node to another.

This article explains the intended specification, the experiment setup, and what we learned.



## Target design (intended specification)

### Roles

* **Always-on front (proxy/standby)**
  Stays up, can issue WoL, and can temporarily “hold” the service identity (MAC/IP) while the target is asleep.
* **Target machine (power-saving)**
  Normally suspended. When woken up, it acquires the service identity and starts serving requests.
* **Clients**
  Always connect to a fixed destination `ip0:PORT`. They do not know or care whether the target is asleep or awake.

### Key idea

Prepare a dedicated “service identity”:

* **Service IP**: `ip0` (e.g., `10.200.0.100`)
* **Service MAC**: `mac0` (e.g., `02:00:00:00:00:64`)

Then, **move (handoff)** that IP/MAC between the standby side and the real server side.

If this works, clients can always connect to `ip0:PORT`, and the backend can dynamically move the identity to the machine that is currently responsible for serving traffic.



## Why this is non-trivial (what must hold true)

This approach depends on the following behaviors being reliable:

1. **Uniqueness**: only one node holds `ip0/mac0` at any time.
2. **TCP behavior during wake-up**: while the target is waking, clients should ideally “wait” rather than immediately fail.
3. **L2/L3 convergence**: switches/bridges and clients must update their view of “where mac0 lives” (FDB learning) and “which MAC corresponds to ip0” (ARP cache).

The pre-experiment is designed specifically to test these constraints in a controlled environment.



## Pre-experiment: simulate the network with Linux namespaces

To make the experiment observable and reproducible, I used Linux `network namespaces` to simulate three hosts:

* `A`, `B`, `C` are separate namespaces (acting like separate machines)
* `veth` pairs connect each namespace into a shared Linux bridge `br-abc` (a single L2 segment)
* A “service identity” (`mac0` + `ip0`) is attached via `macvlan`
* Client namespace `C` starts a TCP connection attempt to `ip0:8080`
* During the attempt, the service identity is moved from `A` to `B`
* `B` runs a server (`nc -l`) to accept the connection once the handoff completes

Conceptually:

```
          (root namespace)
               br-abc (bridge)
           /        |        \
      vethA-br  vethB-br  vethC-br
         |         |         |
       [A]       [B]       [C]
       vethA     vethB     vethC

Service identity ip0/mac0 is attached via macvlan (macv0).
Initially held by A; then moved to B.
```



## The experiment script (key points)

Below is the script used in the experiment (as provided). The flow is:

1. Create namespaces, veth pairs, and a Linux bridge
2. Assign fixed IPs to `A`, `B`, and `C`
3. Set ARP-related sysctls to avoid ARP flux
4. Create `macvlan` in `A`, set `mac0` and `ip0`
5. On `A`, drop incoming TCP to `ip0:8080` so it does not respond
6. From `C`, start a TCP connect attempt to `ip0:8080`
7. Start a server on `B`
8. After a short delay, delete `A`’s `macvlan`
9. Create `macvlan` on `B` and assign the same `mac0` and `ip0`
10. Confirm the client attempt completes

```bash
#!/usr/bin/env bash
set -euo pipefail
set -x

BR=br-abc
MAC0="02:00:00:00:00:64"
IP0_ADDR="10.200.0.100"
IP0_CIDR="10.200.0.100/24"
PORT=8080

# cleanup
ip netns del A 2>/dev/null || true
ip netns del B 2>/dev/null || true
ip netns del C 2>/dev/null || true
ip link del $BR 2>/dev/null || true

ip netns add A
ip netns add B
ip netns add C

ip link add $BR type bridge
ip link set $BR up

ip link add vethA type veth peer name vethA-br
ip link set vethA netns A
ip link set vethA-br master $BR
ip link set vethA-br up

ip link add vethB type veth peer name vethB-br
ip link set vethB netns B
ip link set vethB-br master $BR
ip link set vethB-br up

ip link add vethC type veth peer name vethC-br
ip link set vethC netns C
ip link set vethC-br master $BR
ip link set vethC-br up

ip netns exec A ip link set lo up
ip netns exec A ip link set vethA up
ip netns exec A ip addr add 10.200.0.11/24 dev vethA

ip netns exec B ip link set lo up
ip netns exec B ip link set vethB up
ip netns exec B ip addr add 10.200.0.12/24 dev vethB

ip netns exec C ip link set lo up
ip netns exec C ip link set vethC up
ip netns exec C ip addr add 10.200.0.13/24 dev vethC

for ns in A B; do
  ip netns exec $ns sysctl -w \
    net.ipv4.conf.all.arp_ignore=1 \
    net.ipv4.conf.default.arp_ignore=1 \
    net.ipv4.conf.all.arp_announce=2 \
    net.ipv4.conf.default.arp_announce=2 >/dev/null
done

# A: macvlan + ip0/mac0
ip netns exec A ip link add macv0 link vethA type macvlan mode bridge
ip netns exec A ip link set macv0 address $MAC0
ip netns exec A ip addr add $IP0_CIDR dev macv0
ip netns exec A ip link set macv0 up

# A: drop ip0:8080 (no SYN-ACK / no RST)
if ip netns exec A iptables -V >/dev/null 2>&1; then
  ip netns exec A iptables -I INPUT -d $IP0_ADDR -p tcp --dport $PORT -j DROP
else
  ip netns exec A nft -f - <<NFT
table inet filter {
  chain input {
    type filter hook input priority 0;
    policy accept;
    ip daddr $IP0_ADDR tcp dport $PORT drop
  }
}
NFT
fi

# C: start curl
ip netns exec C nc -v -z $IP0_ADDR $PORT &
# ip netns exec C bash -c "curl -vvv --retry 10 $IP0_ADDR:$PORT" &
CURL_PID=$!

# B: start server
ip netns exec B nc -v -l -p $PORT  &
# ip netns exec B sh -c "python -m http.server $PORT"  &
SERVER_PID=$!

sleep 10

# A: delete macvlan
ip netns exec A ip link del macv0

# B: macvlan + ip0/mac0
ip netns exec B ip link add macv0 link vethB type macvlan mode bridge
ip netns exec B ip link set macv0 address $MAC0
ip netns exec B ip addr add $IP0_CIDR dev macv0
ip netns exec B ip link set macv0 up

# trigger FDB relearn (optional; commented out)
# ip netns exec B ping -c 1 -I macv0 10.200.0.13 >/dev/null 2>&1 || true
# ip netns exec B sh -c "arping -A -c 1 -I macv0 $IP0_ADDR || true" &

wait $CURL_PID
echo "DONE (server log: /tmp/httpserver-b.log)"

kill -s INT $SERVER_PID
```



## What this experiment demonstrates

### 1) “Floating identity” (ip0/mac0) is viable at L2/L3

When `ip0/mac0` is removed from `A` and attached to `B`, the network can converge so that traffic to `ip0` reaches `B`. This is the foundational behavior needed for an on-demand wake-and-serve architecture.

### 2) Using DROP avoids immediate client failure

If a host has an IP address but nothing is listening on `PORT`, the kernel typically returns **RST** to a SYN (resulting in “Connection refused”). That tends to make clients fail fast.

In this experiment, `A` intentionally **drops** TCP to `ip0:8080`, which prevents both:

* SYN-ACK (no server)
* RST (no refusal)

As a result, the client side can continue retrying (either at the TCP layer via retransmits or at the application layer via `curl --retry`) until `B` takes over and starts serving.

This matches the desired production behavior: “while the target is waking, clients wait rather than fail immediately.”

### 3) ARP flux must be controlled

The sysctls:

* `arp_ignore=1`
* `arp_announce=2`

reduce incorrect ARP replies and incorrect source-IP announcements when multiple interfaces exist. Floating IP/MAC designs are especially sensitive to this class of failure, so treating these settings as a baseline requirement is reasonable.



## Operational requirements implied by this design

Based on the experiment, a production-grade design should specify the following as explicit requirements:

### Requirement A: Single ownership of ip0/mac0

Only one node may own the service identity at any time. The takeover sequence must be:

1. old owner releases identity
2. new owner acquires identity

Anything else risks ARP conflicts and non-deterministic delivery.

### Requirement B: Define behavior during wake-up (DROP vs fast fail)

If the system wants “wait until ready” semantics, then the standby side should avoid generating RST. Practically:

* Drop inbound TCP to `ip0:PORT` during the wake-up window, or
* Provide a controlled response strategy that aligns with client behavior (e.g., a proxy, a queue, or an explicit error with retry hints)

The experiment validates the DROP-based approach for keeping clients from failing fast.

### Requirement C: Force L2/L3 cache convergence after handoff

In real networks, MAC learning tables (FDB) and ARP caches may take time to update. A robust implementation typically sends a **gratuitous ARP (GARP)** immediately after takeover, e.g.:

* `arping -A -I <iface> <ip0>`

The script includes this as a commented-out option. In production, this should be treated as a standard part of takeover.



## Mapping to the Suspend + WoL system

Translate namespaces to the real system:

* **A (standby/front)**: always-on node

  * Holds `ip0/mac0` while the target is suspended
  * Drops TCP to `ip0:PORT` to avoid fast-fail
  * Sends WoL when demand is detected
* **B (target machine)**: the suspended machine

  * On boot/resume: acquires `ip0/mac0`, sends GARP, starts the service
* **C (client)**: unchanged

  * Always connects to `ip0:PORT`

This creates a clear contract: clients use a stable endpoint, while the backend can dynamically wake and take over.



## Next validation items

This pre-experiment confirms correctness at a functional level, but production hardening should validate:

* wake-up latency distribution (Suspend → link-up → service-ready)
* client retry strategy and timeouts
* behavior on real switches (MAC learning, port security, aging timers)
* IPv6 implications (NDP, DAD, dual-stack binding behavior)
* handling of existing connections (this experiment focuses on new connects)



## Summary

The experiment shows that **moving a fixed service IP/MAC (ip0/mac0) between nodes** can allow TCP connection attempts to eventually succeed after a takeover—an essential building block for a **Suspend + WoL on-demand access** architecture.

The key design points are:

* enforce single ownership of the service identity
* avoid fast-fail during wake-up (DROP-based waiting semantics)
* trigger cache convergence (GARP) after takeover
* control ARP behavior to prevent ARP flux

With these requirements formalized, the design can be progressed from a namespace-level simulation to an implementation on real hardware.

