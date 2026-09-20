---
title: "Running dnsmasq as a Dedicated DNS Server with macvlan to Avoid systemd-resolved Conflicts"
description: "Running a local DNS service on a modern Linux host can be deceptively tricky. On many distributions,..."
pubDatetime: 2026-01-19T11:18:37.916Z
---

Running a local DNS service on a modern Linux host can be deceptively tricky. On many distributions, `systemd-resolved` already manages name resolution and may bind to port 53 or otherwise influence how DNS traffic is handled. If you start `dnsmasq` directly on the host, you can run into port-binding collisions, unexpected resolver behavior, or hard-to-debug interference between the two components.

A clean way to avoid these conflicts is to run `dnsmasq` as a dedicated DNS server on its own IP address, isolated from the host resolver stack. This setup uses a separate Linux network namespace and a `macvlan` interface, allowing `dnsmasq` to operate independently while keeping `systemd-resolved` enabled on the host.

## Problem Overview: dnsmasq vs. systemd-resolved

When `dnsmasq` runs on the host network stack, it competes with `systemd-resolved` for DNS-related resources:

* **Port 53 conflicts:** Both services may attempt to bind to UDP/TCP port 53.
* **Resolver interference:** The host’s resolution path (e.g., `/etc/resolv.conf`, stub resolvers, and resolver policies) can override or conflict with `dnsmasq` expectations.
* **Operational disruption:** Disabling or reconfiguring `systemd-resolved` to “make room” for `dnsmasq` can have side effects on other services.

The goal is coexistence: keep the host’s resolver untouched, while providing a separate, reliable DNS server endpoint.

## Architecture: Isolate dnsmasq with a Network Namespace and macvlan

The approach creates:

* A **dedicated network namespace** (e.g., `dnsmasq`) to isolate networking for the DNS service.
* A **macvlan interface** attached to a parent interface (e.g., `eth0`) that provides:

  * Its own **MAC address**
  * Its own **IP address**
  * Its own routing (default gateway)

Because the DNS service is bound to an IP address that does not belong to the host’s main network stack, it avoids the typical conflicts with `systemd-resolved`. The host can continue using `systemd-resolved`, while clients on the network can query `dnsmasq` at the dedicated IP.

## dnsmasq Configuration Highlights

The provided `dnsmasq.conf` is intentionally minimalist and focused on authoritative, local answers rather than inheriting host resolver state.

### Interface Binding and Isolation

* `bind-interfaces` ensures `dnsmasq` binds only to available interfaces (in this case, the interface inside the namespace), rather than wildcard-binding in a way that could create conflicts.

### Avoid Using Host Resolver Inputs

These settings prevent `dnsmasq` from consuming host-level resolution sources:

* `no-resolv`, `no-poll`: do not read or watch `/etc/resolv.conf`
* `no-hosts`: do not read `/etc/hosts`

This reinforces the “dedicated DNS server” model: the service is defined by its own configuration, not by the host’s resolver stack.

### Local Zones and Static Records

The example shows defining a local domain and mapping it explicitly:

* `local=/yout.host.name/`
* `address=/yout.host.name/1.2.3.4`

It also configures a reverse DNS zone and PTR record:

* `local=/4.3.2.1.in-addr.arpa/`
* `ptr-record=4.3.2.1.in-addr.arpa,prelude.ddns.net`

Finally, it enables common safety checks:

* `domain-needed`: requires dots in names unless configured otherwise
* `bogus-priv`: blocks forwarding for private reverse lookups

## Service Management with systemd

The deployment uses a templated systemd unit, `dnsmasq-netns.service.template`, which delegates lifecycle steps to a helper script:

* `ExecStartPre=.../run.sh setup`
* `ExecStart=.../run.sh run`
* `ExecStopPost=.../run.sh cleanup`

Key operational properties include:

* `Restart=on-failure` with a short retry delay
* `KillMode=control-group` to properly terminate the full process group
* A bounded stop timeout (`TimeoutStopSec=10`) to avoid hangs

This design keeps the systemd unit simple and pushes networking complexity into a script that can be tested independently.

## Environment-Driven Configuration

The `env.sh` file centralizes deployment-specific details:

* Namespace name: `NS_NAME=dnsmasq`
* Parent interface: `PARENT_IF=eth0`
* macvlan interface name: `MACVLAN_IF=macvlan0`
* Addressing and routing: `MAC_ADDRESS`, `IP_ADDRESS`, `DEFAULT_GW`
* dnsmasq paths and identity: `DNSMASQ_CONF`, `DNSMASQ_BIN`, `DNSMASQ_USER`, `DNSMASQ_GROUP`

This makes the system portable: the logic remains constant while environment values adapt to different hosts or networks.

## Installation Flow

The `install.sh` script performs a straightforward system installation:

1. Validates root execution.
2. Copies configuration into `/etc/dnsmasq-netns`.
3. Installs the runtime script to `/usr/libexec/dnsmasq-netns`.
4. Generates the final systemd service file using `envsubst`.
5. Reloads systemd and checks service status.

This creates a repeatable installation path that avoids hand-editing system unit files.

## The Core Runtime Script: setup, run, cleanup

The `run.sh` script is the operational center. It implements three commands.

### setup: Build the Namespace and Network Interface

The `setup` command:

* Creates the network namespace.
* Creates a `macvlan` interface in bridge mode attached to the parent interface.
* Moves the `macvlan` interface into the namespace.
* Brings up loopback and the `macvlan` link within the namespace.
* Assigns the configured MAC address and IP address.
* Installs a default route via the specified gateway.

This results in a fully functional network stack for the namespace, with its own L2 identity and L3 address.

### run: Start dnsmasq Inside the Namespace with Least Privilege

The `run` command starts `dnsmasq` inside the namespace using:

* `nsenter --net=/run/netns/$NS_NAME` to enter the namespace’s network context.
* `setpriv` to drop privileges and run as `DNSMASQ_USER`/`DNSMASQ_GROUP`.
* Linux capabilities management to grant only what is needed to bind to privileged port 53:

  * `net_bind_service` is retained while others are dropped.

`dnsmasq` is run in the foreground (`--no-daemon`) with query logging enabled, making it systemd-friendly and operationally transparent.

### cleanup: Remove the Interface and Namespace

The `cleanup` command attempts to delete:

* The namespace-local `macvlan` link (if present)
* The namespace itself

This supports idempotent restarts and ensures failed runs do not leave stale namespaces behind.

## Outcome: Clean Coexistence with systemd-resolved

With this model:

* `dnsmasq` runs as a **true dedicated DNS server** bound to a separate IP.
* The host keeps `systemd-resolved` enabled and continues using its standard resolver behavior.
* DNS traffic to `dnsmasq` is naturally separated by IP and namespace boundaries.
* The service is manageable via systemd, reproducible via install scripts, and hardened via least-privilege execution.

This provides a practical, low-disruption solution for operators who need `dnsmasq` functionality without dismantling the host’s default resolution stack.

## `dnsmasq-netns.service.template`

```ini
[Unit]
Description=dnsmasq in netns
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
# WorkingDirectory=
EnvironmentFile=${ETC_DIR_ESC}/env.sh
ExecStartPre=${LIBEXEC_DIR_ESC}/run.sh setup
ExecStart=${LIBEXEC_DIR_ESC}/run.sh run
ExecStopPost=${LIBEXEC_DIR_ESC}/run.sh cleanup
KillMode=control-group
TimeoutStopSec=10
Restart=on-failure
RestartSec=2

[Install]
WantedBy=multi-user.target
```

## `dnsmasq.conf`
```
bind-interfaces

no-resolv
no-poll
no-hosts

local=/yout.host.name/

address=/yout.host.name/1.2.3.4

local=/4.3.2.1.in-addr.arpa/
ptr-record=4.3.2.1.in-addr.arpa,prelude.ddns.net

domain-needed
bogus-priv
```

## `env.sh`

```bash
NS_NAME=dnsmasq
PARENT_IF=eth0
MACVLAN_IF=macvlan0
MAC_ADDRESS=...
IP_ADDRESS=...
DEFAULT_GW=...

DNSMASQ_CONF=/etc/dnsmasq-netns/dnsmasq.conf
DNSMASQ_BIN=/usr/sbin/dnsmasq
DNSMASQ_USER=...
DNSMASQ_GROUP=...
```

## `install.sh`

```bash
#!/bin/bash

if [[ "${EUID:-$(id -u)}" -ne 0 ]]; then
  echo "error: execute as root" >&2
  exit 1
fi

set -xe

ETC_DIR=/etc/dnsmasq-netns
mkdir -p "$ETC_DIR"
cp -t "$ETC_DIR" dnsmasq.conf env.sh

LIBEXEC_DIR=/usr/libexec/dnsmasq-netns
mkdir -p "$LIBEXEC_DIR"
cp -t "$LIBEXEC_DIR" run.sh

SYSTEMD_DIR=/etc/systemd/system
export ETC_DIR_ESC="$(printf "%q" "$ETC_DIR")"
export LIBEXEC_DIR_ESC="$(printf "%q" "$LIBEXEC_DIR")"
envsubst < dnsmasq-netns.service.template > "$SYSTEMD_DIR"/dnsmasq-netns.service
systemctl daemon-reload

systemctl status dnsmasq-netns
```

## `run.sh`

```bash
#!/bin/bash
set -euo pipefail

log() { echo "[netns-dnsmasq] $*" >&2; }

need() {
  local v="$1"
  if [[ -z "${!v:-}" ]]; then
    echo "Required variable not set: $v" >&2
    exit 2
  fi
}

: "${DNSMASQ_BIN:=/usr/sbin/dnsmasq}"

setup() {
  need NS_NAME
  need PARENT_IF
  need MACVLAN_IF
  need MAC_ADDRESS
  need IP_ADDRESS
  need DEFAULT_GW
  need DNSMASQ_CONF

  [[ -f "$DNSMASQ_CONF" ]] || { echo "dnsmasq.conf not found: $DNSMASQ_CONF" >&2; exit 2; }

  ip link show dev "$PARENT_IF" >/dev/null

  if ip netns list | awk '{print $1}' | grep -qx "$NS_NAME"; then
    log "netns $NS_NAME already exists; cleaning up first"
    cleanup || true
  fi

  log "create netns: $NS_NAME"
  ip netns add "$NS_NAME"

  log "create macvlan: $MACVLAN_IF on $PARENT_IF"
  ip link add "$MACVLAN_IF" link "$PARENT_IF" type macvlan mode bridge

  log "move macvlan to netns: $NS_NAME"
  ip link set "$MACVLAN_IF" netns "$NS_NAME"

  log "bring up lo in netns"
  ip -n "$NS_NAME" link set lo up

  log "set MAC address: $MAC_ADDRESS"
  ip -n "$NS_NAME" link set dev "$MACVLAN_IF" address "$MAC_ADDRESS"

  log "assign IP: $IP_ADDRESS"
  ip -n "$NS_NAME" addr add "$IP_ADDRESS" dev "$MACVLAN_IF"

  log "link up: $MACVLAN_IF"
  ip -n "$NS_NAME" link set dev "$MACVLAN_IF" up

  log "set default route via $DEFAULT_GW"
  ip -n "$NS_NAME" route replace default via "$DEFAULT_GW" dev "$MACVLAN_IF"
}

run() {
  need NS_NAME
  need DNSMASQ_CONF
  need DNSMASQ_BIN

  exec nsenter --net=/run/netns/"$NS_NAME" \
       setpriv \
       --reuid="$DNSMASQ_USER" --regid="$DNSMASQ_GROUP" --init-groups \
       --bounding-set=-all,+net_bind_service \
       --inh-caps=+net_bind_service \
       --ambient-caps=+net_bind_service \
       "$DNSMASQ_BIN" \
       --no-daemon \
       --log-queries --log-facility=- \
       --conf-file="$DNSMASQ_CONF"

}

cleanup() {
  if [[ -n "${NS_NAME:-}" ]] && ip netns list | awk '{print $1}' | grep -qx "$NS_NAME"; then
    if [[ -n "${MACVLAN_IF:-}" ]]; then
      ip -n "$NS_NAME" link del "$MACVLAN_IF" 2>/dev/null || true
    fi
    ip netns del "$NS_NAME" 2>/dev/null || true
    log "deleted netns: $NS_NAME"
  fi
}

cmd="${1:-}"
case "$cmd" in
  setup)   setup ;;
  run)     run ;;
  cleanup) cleanup ;;
  *)
    echo "Usage: $0 {setup|run|cleanup}" >&2
    exit 2
    ;;
esac
```
