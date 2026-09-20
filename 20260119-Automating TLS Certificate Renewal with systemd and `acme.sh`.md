---
title: "Automating TLS Certificate Renewal with systemd and `acme.sh`"
description: "Schedule ACME webroot validation with systemd, temporarily forward port 80 through UPnP, and deploy renewed certificates to Nginx and StrongSwan."
pubDatetime: 2026-01-19T10:12:31.719Z
---

## How the Setup Works

This configuration renews TLS certificates automatically using a `systemd` timer and an `acme.sh`-based deployment script. A `cert-deploy.timer` runs twice daily with a randomized delay, triggering `cert-deploy.service`, which executes `cert-deploy.sh` under a controlled environment file.

The script must run as root, detects the machine’s local IP via `ip -j route get` and `jq`, then temporarily opens WAN port 80 to LAN port 80 using UPnP for ACME HTTP validation. It calls `acme.sh --issue --webroot` with a normalized domain list and RSA 4096-bit keys. If renewal is needed, it installs the new certificate files and reloads services safely by running `nginx -t`, restarting NGINX, and restarting StrongSwan.

* Timer-based execution (00:00 and 12:00) with jitter and persistence
* UPnP port open/close with cleanup via `trap`
* Webroot validation for one or multiple domains
* Atomic reload trigger using a temporary flag file
* Simple installer to place units, scripts, and enable the timer

## `cert-deploy.service`

```ini
[Unit]
Description=cert & deploy
After=multi-user.target

[Service]
Type=simple
ExecStart=/usr/libexec/acmebot/cert-deploy.sh
EnvironmentFile=/etc/acmebot/env
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

CERTPATH="${ACME_HOME}/${MAIN_DOMAIN}"

local_ip=""

cleanup() {
  # Always close port even on failure
  echo "[info] upnp port close" 1>&2
  sudo -u"$EXECUTE_USER" -H upnpc -d 80 tcp || true
}
trap cleanup EXIT

# jq is required for this block
echo "[info] getting local address (ip -j route get + jq)" 1>&2
if ! command -v jq >/dev/null 2>&1; then
  echo "[error] jq is not installed. Exiting." 1>&2
  exit 1
fi

local_ip="$(ip -j route get 1.1.1.1 | jq -r '.[0].prefsrc // empty')"
if [ -z "$local_ip" ] || [ "$local_ip" = "null" ]; then
  echo "[error] failed to detect local_ip via ip -j route get" 1>&2
  exit 1
fi
echo "[info] local_ip=${local_ip}" 1>&2

# NOTE: This opens WAN:80 -> LAN:80.
echo "[info] upnp port open (WAN 80 -> LAN 80)" 1>&2
sudo -u"$EXECUTE_USER" -H upnpc -a "${local_ip}" 80 80 tcp 600

echo "[info] acme.sh (webroot)" 1>&2

# Check for the existence of acme.sh
if [ ! -x "$ACME_SH" ]; then
  echo "[error] acme.sh not found: $ACME_SH" 1>&2
  echo "[hint] install: curl https://get.acme.sh | sh -s email=$EMAIL" 1>&2
  exit 1
fi

# Expand domain arguments into -d ...
# Normalize DOMAINLIST so that it works whether it is "a,b,c" or "a b c"
DOMAIN_ARGS=()
while read -r d; do
  [ -n "$d" ] && DOMAIN_ARGS+=("-d" "$d")
done < <(echo "$DOMAINLIST" | tr ', ' '\n' | sed '/^$/d')

# Issue (or renew):
#   --webroot for validation
#   --home to fix the acme.sh home under LEPATH
#   --accountemail for the account email
#   --keylength for RSA 4096
#
# Important:
#   If certificates already exist, they will only be renewed when necessary
#   (this is acme.sh behavior)
set +e
sudo -u"$EXECUTE_USER" -H "$ACME_SH" \
  --home "$ACME_HOME" \
  --accountemail "$EMAIL" \
  --issue --webroot "$WEBROOT" \
  --keylength 4096 \
  "${DOMAIN_ARGS[@]}"
ACME_STATUS=$?
set -e

if [ $ACME_STATUS -eq 2 ]; then
  echo "[info] acme.sh exited with code $ACME_STATUS (renew not necessary)" 1>&2
  exit
fi

if [ $ACME_STATUS -ne 0 ]; then
  echo "[warn] acme.sh exited with code $ACME_STATUS" 1>&2
  exit $ACME_STATUS
fi

# Deployment destinations (aligned with the original script paths)
NGINX_CERT_DIR="/etc/nginx/pki/certs"
IPSEC_PRIV="/etc/ipsec.d/private/privkey.pem"
IPSEC_CACERT="/etc/ipsec.d/cacerts/chain.pem"
IPSEC_CERT="/etc/ipsec.d/certs/fullchain.pem"

echo "[info] install certs (acme.sh --install-cert)" 1>&2

RELOAD_FLAG="/tmp/.reload-flag-$(date "+%Y%m%d%H%M%S%N")"

# acme.sh install is normally executed only when a renewal occurs
# (intended to be called after --issue)
# Bundle restarts/tests into --reloadcmd so that failures cause acme.sh to fail
"$ACME_SH" \
  --home "$ACME_HOME" \
  --install-cert -d "$MAIN_DOMAIN" \
  --fullchain-file "${NGINX_CERT_DIR}/fullchain.pem" \
  --key-file       "${NGINX_CERT_DIR}/privkey.pem" \
  --ca-file        "${IPSEC_CACERT}" \
  --reloadcmd "touch '${RELOAD_FLAG}'"

if [[ -f "${RELOAD_FLAG}" ]]; then
  nginx -t
  systemctl restart nginx

  systemctl restart strongswan-starter

  rm "${RELOAD_FLAG}"
fi


echo "[info] done" 1>&2
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
``````

env.sample
``````
EXECUTE_USER=acmebot
DOMAINLIST=hoge.ddns.net
MAIN_DOMAIN=hoge.ddns.net
EMAIL=hoge@mail.com
ACME_HOME=/var/lib/acmebot/acme.sh
ACME_SH=/usr/libexec/acmebot/acme.sh
WEBROOT=/var/lib/acmebot/webroot
```

## `install.sh`

```bash
#!/bin/bash

# Must run as root
if (( EUID != 0 )); then
  echo "This script must be run as root. Exiting."
  exit 1
fi

set -xe

. ./env
mkdir -p --mode=0755 "${WEBROOT}"

mkdir -p --mode=0755 /etc/acmebot
install --mode=0400 ./env /etc/acmebot/

mkdir -p --mode=0755 /usr/libexec/acmebot/
install --mode=0555 ./cert-deploy.sh /usr/libexec/acmebot/

if [[ ! -f "${ACME_SH}" ]]; then
    curl -o "${ACME_SH}" https://get.acme.sh
    chmod 0755 "${ACME_SH}"
fi

install --mode=0644 cert-deploy.service /etc/systemd/system/
install --mode=0644 cert-deploy.timer /etc/systemd/system/

systemctl daemon-reload

systemctl enable cert-deploy.timer
systemctl start cert-deploy.timer
```
