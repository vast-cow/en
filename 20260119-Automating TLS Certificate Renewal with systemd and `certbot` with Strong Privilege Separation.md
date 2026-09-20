---
title: "Automating TLS Certificate Renewal with systemd and `certbot` with Strong Privilege Separation"
description: "Automate certificate issuance under an unprivileged account while reserving deployment for root, with temporary UPnP access, renewal hooks, and validation before service restarts."
pubDatetime: 2026-01-19T09:58:40.606Z
updatedDate: 2026-01-19T10:03:38.984Z
---

This setup automates Let’s Encrypt certificate renewal with `certbot`, scheduled by `systemd`, while emphasizing security through minimal privilege and controlled escalation. Root is used only where unavoidable (service management and writing into protected certificate locations); network-facing and application-level actions are performed as an unprivileged account.

## Security Model and Privilege Boundaries

### Run as root only for system-level actions

The workflow is executed by a `systemd` service and the script enforces `EUID==0`. Root is required to:

* write certificates into privileged directories (e.g., Nginx and StrongSwan locations)
* restart services via `systemctl`
* perform final configuration validation and deployment steps

Everything else is structured to avoid running as root.

### Drop privileges for cert issuance and UPnP operations

The script uses a dedicated unprivileged user (configured as `EXECUTE_USER`, e.g., `acmebot`) and explicitly runs high-risk or externally influenced operations as that user:

* UPnP port mapping commands (`upnpc`) run with `sudo -u"$EXECUTE_USER" -H ...`
* `certbot` runs from a virtual environment under the unprivileged user:

  * reduces impact if `certbot`, Python packages, plugins, or parsing logic are compromised
  * keeps ACME client behavior out of the root context

This design prevents network-adjacent tooling from running with full system privileges.

## systemd Units: Minimal and Predictable Execution

### Service unit

`cert-deploy.service` launches the renewal/deploy script and loads configuration via an `EnvironmentFile`. It also enables `PrivateTmp=true`, limiting exposure and interference through shared temporary paths.

### Timer unit

`cert-deploy.timer` triggers execution twice daily and adds `RandomizedDelaySec=1h`, which helps avoid synchronized renewal bursts. `Persistent=true` ensures missed runs (e.g., downtime) are executed on the next boot.

## Deployment Logic Designed to Avoid Unnecessary Risk

### Only restart when renewal actually occurs

A unique runtime flag file is created and only “touched” via `certbot`’s `--deploy-hook` when renewal/issuance happens. If the flag is absent, deployment and restarts are skipped. This reduces operational churn and avoids restarting critical services without need.

### Validate certificate artifacts before use

Before copying, the script checks that `fullchain.pem`, `privkey.pem`, and `chain.pem` exist and are non-empty. This prevents propagating broken or partial state into production paths.

### Validate Nginx before restart

`nginx -t` is executed prior to restarting Nginx, reducing outage risk from misconfiguration during the deployment window.

## Controlled Exposure for HTTP-01 Validation

For environments behind consumer routers, the script temporarily opens WAN:80 → LAN:80 via UPnP to satisfy HTTP-01 validation, then closes it using an `EXIT` trap so cleanup occurs even on failure. Running UPnP actions as the unprivileged user reduces the blast radius if those tools behave unexpectedly.

## Installation Hardening and Dependency Isolation

The install process:

* verifies required tools (`iproute2` JSON output, `upnpc`, `jq`, `python3`) and fails fast if missing
* creates a dedicated `certbot` virtual environment (`CERTBOT_VENV`) instead of installing into the system Python environment
* applies ownership so the unprivileged user can write only to the directories it needs (config/work/log/webroot), while root retains control over system-wide deployment targets

## Practical Security Notes

* Use a dedicated, non-login service account for `EXECUTE_USER`, with no shell and minimal group memberships.
* Keep the environment file (`/etc/acmebot-certbot/env`) root-owned and non-world-readable, since it contains operational paths and identifiers.
* Limit write permissions strictly: the unprivileged user should not be able to write into Nginx/StrongSwan certificate directories directly; root performs the final copy.
* Consider tightening the systemd unit further with additional sandboxing (e.g., filesystem and capability restrictions) if you want stronger containment beyond `PrivateTmp`.

This design achieves automated renewals while keeping network-adjacent tooling and ACME client execution out of the root context, reserving elevated privileges for only the final deployment and service lifecycle operations.

## `cert-deploy.service`

```ini
[Unit]
Description=cert & deploy
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/libexec/acmebot-certbot/cert-deploy.sh
EnvironmentFile=/etc/acmebot-certbot/env
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

## `cert-deploy.sh`

```bash
#!/bin/bash

# Must run as root
if (( EUID != 0 )); then
  echo "This script must be run as root. Exiting."
  exit 1
fi

set -xe
set -o pipefail

FLAGFILE="/run/acmebot-certbot/deploy-flag.$(date "+%Y%m%d%H%M%S%N")"
CERTPATH="$CONFIG_PATH/live/$CERT_NAME"

local_ip=""

cleanup() {
  # Always close port even on failure
  echo "[info] upnp port close" 1>&2
  sudo -u"$EXECUTE_USER" -H upnpc -d 80 tcp || true
}
trap cleanup EXIT

echo "[info] getting local address (ip -j route get + jq)" 1>&2

# ip -j route get 1.1.1.1 returns a JSON array; take the first element's "prefsrc"
local_ip="$(ip -j route get 1.1.1.1 | jq -r '.[0].prefsrc // empty')"

if [ -z "$local_ip" ] || [ "$local_ip" = "null" ]; then
  echo "[error] failed to detect local_ip via ip -j route get" 1>&2
  exit 1
fi

echo "[info] local_ip=${local_ip}" 1>&2

# NOTE: This opens WAN:80 -> LAN:80. If you intend WAN:80 -> LAN:8080, change the first "80" to "8080".
echo "[info] upnp port open (WAN 80 -> LAN 80)" 1>&2
sudo -u"$EXECUTE_USER" -H upnpc -a "${local_ip}" 80 80 tcp 600

echo "[info] certbot (webroot)" 1>&2

# Run certbot and capture exit code without aborting the script immediately
set +e
sudo -u "$EXECUTE_USER" -H \
     "${CERTBOT_VENV}/bin/certbot" \
     certonly -q --webroot -w "$WEB_ROOT" \
     --cert-name "${CERT_NAME}" \
     --key-type rsa --rsa-key-size 4096 \
     -d "$DOMAINLIST" \
     --agree-tos --no-eff-email --email "$EMAIL" \
     --deploy-hook "touch '$FLAGFILE'" \
     --config-dir "$CONFIG_PATH" \
     --work-dir "$WORK_DIR" \
     --logs-dir "$LOGS_DIR"
CERTBOT_STATUS=$?
set -e

if [ $CERTBOT_STATUS -ne 0 ]; then
  echo "[warn] Certbot exited with code $CERTBOT_STATUS" 1>&2
  exit $CERTBOT_STATUS
fi

if [ ! -e "$FLAGFILE" ]; then
  echo "[info] cert not renewed. skip deploy/restart." 1>&2
  exit 0
fi

echo "[info] cert renewed. proceed deploy/restart." 1>&2
rm -f "$FLAGFILE"

# Ensure cert files exist and are non-empty
for f in fullchain.pem privkey.pem chain.pem; do
  if [ ! -s "${CERTPATH}/${f}" ]; then
    echo "[error] missing or empty ${CERTPATH}/${f}" 1>&2
    exit 2
  fi
done

echo "[info] put certs" 1>&2

# nginx
cp -t "${NGINX_CERTS}" "${CERTPATH}/fullchain.pem" "${CERTPATH}/privkey.pem"

# strongswan
cp "${CERTPATH}/privkey.pem" "${IPSEC_ETC}"/private/privkey.pem
cp "${CERTPATH}/chain.pem" "${IPSEC_ETC}"/cacerts/chain.pem
cp "${CERTPATH}/fullchain.pem" "${IPSEC_ETC}"/certs/fullchain.pem

echo "[info] nginx config test" 1>&2
nginx -t

echo "[info] restart nginx and strongswan" 1>&2
systemctl restart nginx
systemctl restart strongswan-starter
```

## `cert-deploy.timer`

```ini
[Unit]
Description=Run certbot_update_cert periodically

[Timer]
OnCalendar=*-*-* 00:00:00
OnCalendar=*-*-* 12:00:00
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

## `env.sample`

```bash
EXECUTE_USER=acmebot

ETC_DIR=/etc/acmebot-certbot
LIBEXEC_DIR=/usr/libexec/acmebot-certbot
SYSTEMD_DIR=/etc/systemd/system
RUN_DIR=/run/acmebot-certbot

CERT_NAME=hoge.ddns.net
DOMAINLIST=hoge.ddns.net
EMAIL=hoge@mail.com

CERTBOT_VENV=/usr/libexec/acmebot-certbot/venv

CONFIG_PATH=/var/lib/acmebot-certbot/config
WORK_DIR=/run/acmebot-certbot/work
LOGS_DIR=/var/log/acmebot-certbot

WEB_ROOT=/var/lib/acmebot-certbot/webroot

NGINX_CERTS=/etc/nginx/pki/certs

IPSEC_ETC=/etc/ipsec.d
```

## `install.sh`

```bash
#!/bin/bash

# Must run as root
if (( EUID != 0 )); then
  echo "[error] This script must be run as root. Exiting."
  exit 1
fi

set -xe

missing=()

# Check if `ip -j address` runs successfully
if ! ip -j address >/dev/null 2>&1; then
    missing+=("ip -j address (iproute2)")
fi

# Check if upnpc exists
if ! command -v upnpc >/dev/null 2>&1; then
    missing+=("upnpc")
fi

# Check if jq exists
if ! command -v jq >/dev/null 2>&1; then
    missing+=("jq")
fi

# Check if python3 exists
if ! command -v python3 >/dev/null 2>&1; then
    missing+=("python3")
fi

# Final check
if [ "${#missing[@]}" -ne 0 ]; then
    echo "The following requirements are missing or not working:"
    for item in "${missing[@]}"; do
        echo " - $item"
    done
    exit 1
fi

. ./env

mkdir -p "$ETC_DIR"
cp -t "$ETC_DIR" ./env

mkdir -p "${RUN_DIR}"
chown "${EXECUTE_USER}" "${RUN_DIR}"

mkdir -p "${LIBEXEC_DIR}"
install --mode=0755 cert-deploy.sh "${LIBEXEC_DIR}"

if [[ ! -x "${CERTBOT_VENV}/bin/certbot" ]]; then
    if [[ ! -x "${CERTBOT_VENV}/bin/python" ]]; then
        if [[ ! -d "${CERTBOT_VENV}" ]]; then
            mkdir -p "${CERTBOT_VENV}"
        fi
        python3 -m venv "${CERTBOT_VENV}"
    fi
    "${CERTBOT_VENV}/bin/pip" install cffi==1.17.1 certbot
fi

mkdir -p "${CONFIG_PATH}"
chown "${EXECUTE_USER}" "${CONFIG_PATH}"

mkdir -p "${WORK_DIR}"
chown "${EXECUTE_USER}" "${WORK_DIR}"

mkdir -p "${LOGS_DIR}"
chown "${EXECUTE_USER}" "${LOGS_DIR}"

mkdir -p --mode=0755 "${WEB_ROOT}"
chown "${EXECUTE_USER}" "${WEB_ROOT}"

install --mode=0644 cert-deploy.service "${SYSTEMD_DIR}"
install --mode=0644 cert-deploy.timer "${SYSTEMD_DIR}"
systemctl daemon-reload

systemctl enable cert-deploy.timer
systemctl start cert-deploy.timer
```
