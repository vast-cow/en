---
title: "Automatically Renewing Tailscale HTTPS Certificates on PiKVM"
description: "Renew certificates for PiKVM's own Nginx with a systemd timer that checks validity, validates replacement keys, supports rollback, and restores read-only operation."
pubDatetime: 2026-08-10T07:08:34.109Z
updatedDate: 2026-08-31T14:55:39.031Z
---

It is appropriate to continue using

```nginx
ssl_certificate /etc/kvmd/nginx/ssl/server.crt;
ssl_certificate_key /etc/kvmd/nginx/ssl/server.key;
```

in `/etc/kvmd/nginx/ssl.conf`, with a systemd timer checking the certificate expiration and updating these two files only when necessary.

The official PiKVM documentation also describes placing the Tailscale certificate in `/etc/kvmd/nginx/ssl/server.{crt,key}`, setting the group to `kvmd-nginx`, and then restarting `kvmd-nginx`. ([Pikvm][1]) Also, certificates obtained as files using `tailscale cert` are not automatically renewed, so users need to implement their own renewal process. `--min-validity` is also officially available in the current CLI. ([Tailscale][2])

### Configuration

Normally, the setup looks like this.

```text
Tailscale
   │
   │ 100.x / MagicDNS
   ▼
PiKVM nginx :443
   │
   ├─ /etc/kvmd/nginx/ssl/server.crt
   └─ /etc/kvmd/nginx/ssl/server.key
```

Do not use `tailscale serve`.

```bash
tailscale serve --https=443 off
```

The certificate renewal process will be:

```text
Timer runs once a day
        │
        ▼
Check current server.crt
        │
        ├─ FQDN is correct
        │  and at least 30 days remain
        │       → Do nothing
        │
        └─ Less than 30 days / no certificate / hostname mismatch
                │
                ▼
               rw
                │
                ▼
        tailscale cert
                │
                ▼
        Validate cert/key
                │
                ▼
        Replace nginx files
                │
                ▼
        nginx -t
                │
                ▼
        restart kvmd-nginx
                │
                ▼
               ro
```

Let’s Encrypt certificates are valid for 90 days, so attempting renewal starting 30 days before expiration provides plenty of margin. ([Tailscale][3])

---

## 1. Renewal Script

Create `/usr/local/libexec/pikvm-tailscale-cert-renew`.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

export PATH=/usr/local/bin:/usr/bin

CERT="/etc/kvmd/nginx/ssl/server.crt"
KEY="/etc/kvmd/nginx/ssl/server.key"

# 30 days
MIN_VALIDITY_SECONDS=$((30 * 24 * 60 * 60))
TS_MIN_VALIDITY="720h"

TMP=""
MADE_RW=0


log() {
    echo "pikvm-tailscale-cert-renew: $*"
}


cleanup() {
    rc=$?

    trap - EXIT INT TERM

    rm -f "${CERT}.new" "${KEY}.new" 2>/dev/null || true

    if [[ -n "${TMP:-}" ]]; then
        rm -rf "$TMP"
    fi

    if (( MADE_RW )); then
        sync

        if ! ro; then
            log "ERROR: failed to restore read-only filesystem"
            rc=1
        fi
    fi

    exit "$rc"
}

trap cleanup EXIT INT TERM


#
# Get the Tailscale FQDN
#
DOMAIN="$(
    tailscale status --json |
        jq -er '.Self.DNSName | rtrimstr(".") | select(length > 0)'
)"

log "Tailscale DNS name: ${DOMAIN}"


#
# Check the certificate currently used by nginx
#
cert_is_current() {
    [[ -s "$CERT" ]] || return 1
    [[ -s "$KEY" ]] || return 1

    # Check whether the hostname matches
    openssl x509 \
        -in "$CERT" \
        -noout \
        -checkhost "$DOMAIN" \
        >/dev/null 2>&1 || return 1

    # Check whether at least 30 days remain
    openssl x509 \
        -in "$CERT" \
        -noout \
        -checkend "$MIN_VALIDITY_SECONDS" \
        >/dev/null 2>&1 || return 1

    return 0
}


if cert_is_current; then
    log "certificate is valid for more than 30 days; nothing to do"
    exit 0
fi

log "certificate renewal is required"


#
# Use /tmp for the temporary directory.
# The root filesystem is still RO at this point.
#
TMP="$(mktemp -d /tmp/pikvm-tailscale-cert.XXXXXX)"


#
# Switch the PiKVM root filesystem to RW only when necessary.
#
ROOT_OPTS="$(findmnt -no OPTIONS /)"

case ",${ROOT_OPTS}," in
    *,rw,*)
        log "root filesystem is already read-write"
        ;;
    *)
        log "switching root filesystem to read-write"
        rw
        MADE_RW=1
        ;;
esac


#
# Obtain the certificate from Tailscale.
#
# --min-validity=720h requests a certificate
# that is valid for at least 30 days.
#
log "requesting certificate for ${DOMAIN}"

tailscale cert \
    --min-validity="$TS_MIN_VALIDITY" \
    --cert-file="$TMP/server.crt" \
    --key-file="$TMP/server.key" \
    "$DOMAIN"


#
# Validate the obtained certificate
#

# hostname
openssl x509 \
    -in "$TMP/server.crt" \
    -noout \
    -checkhost "$DOMAIN"

# expiration
openssl x509 \
    -in "$TMP/server.crt" \
    -noout \
    -checkend "$MIN_VALIDITY_SECONDS"

# Verify that the certificate and private key have the same public key
if ! cmp -s \
    <(
        openssl x509 \
            -in "$TMP/server.crt" \
            -pubkey \
            -noout |
        openssl pkey \
            -pubin \
            -outform DER 2>/dev/null
    ) \
    <(
        openssl pkey \
            -in "$TMP/server.key" \
            -pubout \
            -outform DER 2>/dev/null
    )
then
    log "ERROR: certificate and private key do not match"
    exit 1
fi


#
# Back up the current certificate
#
if [[ -e "$CERT" ]]; then
    cp -a "$CERT" "$TMP/old.crt"
fi

if [[ -e "$KEY" ]]; then
    cp -a "$KEY" "$TMP/old.key"
fi


rollback() {
    log "rolling back certificate"

    if [[ -e "$TMP/old.crt" ]]; then
        cp -a "$TMP/old.crt" "$CERT"
    else
        rm -f "$CERT"
    fi

    if [[ -e "$TMP/old.key" ]]; then
        cp -a "$TMP/old.key" "$KEY"
    else
        rm -f "$KEY"
    fi
}


#
# Prepare the files for nginx, then rename them.
#
# nginx itself continues holding the old certificate until it is
# reloaded/restarted, so even if the crt/key files briefly do not match
# between the two renames, this does not affect the running nginx process.
#
install \
    -o root \
    -g kvmd-nginx \
    -m 0644 \
    "$TMP/server.crt" \
    "${CERT}.new"

install \
    -o root \
    -g kvmd-nginx \
    -m 0640 \
    "$TMP/server.key" \
    "${KEY}.new"

mv -f "${KEY}.new" "$KEY"
mv -f "${CERT}.new" "$CERT"


#
# Validate using the actual nginx configuration generated by PiKVM
#
if ! nginx -t -c /run/kvmd/nginx.conf; then
    log "ERROR: nginx configuration test failed"
    rollback
    exit 1
fi


#
# Restart according to the official PiKVM documentation.
#
if ! systemctl restart kvmd-nginx; then
    log "ERROR: kvmd-nginx restart failed"

    rollback

    # Attempt recovery after restoring the old certificate
    nginx -t -c /run/kvmd/nginx.conf || true
    systemctl restart kvmd-nginx || true

    exit 1
fi


log "certificate successfully installed"

openssl x509 \
    -in "$CERT" \
    -noout \
    -subject \
    -issuer \
    -dates

exit 0
```

With this method, the normal daily operation consists only of:

```bash
openssl x509 -checkhost ...
openssl x509 -checkend ...
```

so the **root filesystem remains RO**.

It switches to `rw` only when fewer than 30 days remain.

Additionally, because `tailscale cert --min-validity=720h` is used, Tailscale is also instructed to “return a certificate that is valid for at least 30 days.” This flag is part of the current Tailscale CLI specification. ([Tailscale][2])

---

## 2. systemd Service

`/etc/systemd/system/pikvm-tailscale-cert-renew.service`

```ini
[Unit]
Description=Renew Tailscale TLS certificate for PiKVM nginx
Wants=network-online.target
After=network-online.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=oneshot
ExecStart=/usr/local/libexec/pikvm-tailscale-cert-renew
TimeoutStartSec=5min
```

There is no need to add `kvmd-nginx.service` to `Requires=`.

The reason is that even if `kvmd-nginx` has stopped because of a broken certificate, this unit should still be able to repair the certificate independently and then run `systemctl restart kvmd-nginx`.

---

## 3. systemd Timer

`/etc/systemd/system/pikvm-tailscale-cert-renew.timer`

```ini
[Unit]
Description=Periodic Tailscale TLS certificate check for PiKVM

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
RandomizedDelaySec=30min
AccuracySec=1min
Unit=pikvm-tailscale-cert-renew.service

[Install]
WantedBy=timers.target
```

`Persistent=true` is intentionally omitted here.

Since this configuration starts renewing a 90-day certificate 30 days before expiration, missing a single check while the device is powered off is not a problem. The certificate will be checked roughly 15–45 minutes after boot, and then approximately once per day thereafter.

---

## 4. Installation

Copy and paste the entire block below into a root shell on PiKVM. It installs `jq` first, creates the renewal script and both systemd unit files with the exact contents shown above, disables Tailscale Serve, enables **and starts** the timer, runs the renewal service once immediately, and finally restores the root filesystem to RO.

```bash
(
    set -Eeuo pipefail
    trap 'ro >/dev/null 2>&1 || true' EXIT

    rw

    # Install jq before installing/enabling the renewal service.
    pacman -S --needed jq

    install -d -m 0755 /usr/local/libexec

    cat > /usr/local/libexec/pikvm-tailscale-cert-renew <<'PIKVM_RENEW_EOF'
#!/usr/bin/env bash
set -Eeuo pipefail

export PATH=/usr/local/bin:/usr/bin

CERT="/etc/kvmd/nginx/ssl/server.crt"
KEY="/etc/kvmd/nginx/ssl/server.key"

# 30 days
MIN_VALIDITY_SECONDS=$((30 * 24 * 60 * 60))
TS_MIN_VALIDITY="720h"

TMP=""
MADE_RW=0


log() {
    echo "pikvm-tailscale-cert-renew: $*"
}


cleanup() {
    rc=$?

    trap - EXIT INT TERM

    rm -f "${CERT}.new" "${KEY}.new" 2>/dev/null || true

    if [[ -n "${TMP:-}" ]]; then
        rm -rf "$TMP"
    fi

    if (( MADE_RW )); then
        sync

        if ! ro; then
            log "ERROR: failed to restore read-only filesystem"
            rc=1
        fi
    fi

    exit "$rc"
}

trap cleanup EXIT INT TERM


#
# Get the Tailscale FQDN
#
DOMAIN="$(
    tailscale status --json |
        jq -er '.Self.DNSName | rtrimstr(".") | select(length > 0)'
)"

log "Tailscale DNS name: ${DOMAIN}"


#
# Check the certificate currently used by nginx
#
cert_is_current() {
    [[ -s "$CERT" ]] || return 1
    [[ -s "$KEY" ]] || return 1

    # Check whether the hostname matches
    openssl x509 \
        -in "$CERT" \
        -noout \
        -checkhost "$DOMAIN" \
        >/dev/null 2>&1 || return 1

    # Check whether at least 30 days remain
    openssl x509 \
        -in "$CERT" \
        -noout \
        -checkend "$MIN_VALIDITY_SECONDS" \
        >/dev/null 2>&1 || return 1

    return 0
}


if cert_is_current; then
    log "certificate is valid for more than 30 days; nothing to do"
    exit 0
fi

log "certificate renewal is required"


#
# Use /tmp for the temporary directory.
# The root filesystem is still RO at this point.
#
TMP="$(mktemp -d /tmp/pikvm-tailscale-cert.XXXXXX)"


#
# Switch the PiKVM root filesystem to RW only when necessary.
#
ROOT_OPTS="$(findmnt -no OPTIONS /)"

case ",${ROOT_OPTS}," in
    *,rw,*)
        log "root filesystem is already read-write"
        ;;
    *)
        log "switching root filesystem to read-write"
        rw
        MADE_RW=1
        ;;
esac


#
# Obtain the certificate from Tailscale.
#
# --min-validity=720h requests a certificate
# that is valid for at least 30 days.
#
log "requesting certificate for ${DOMAIN}"

tailscale cert \
    --min-validity="$TS_MIN_VALIDITY" \
    --cert-file="$TMP/server.crt" \
    --key-file="$TMP/server.key" \
    "$DOMAIN"


#
# Validate the obtained certificate
#

# hostname
openssl x509 \
    -in "$TMP/server.crt" \
    -noout \
    -checkhost "$DOMAIN"

# expiration
openssl x509 \
    -in "$TMP/server.crt" \
    -noout \
    -checkend "$MIN_VALIDITY_SECONDS"

# Verify that the certificate and private key have the same public key
if ! cmp -s \
    <(
        openssl x509 \
            -in "$TMP/server.crt" \
            -pubkey \
            -noout |
        openssl pkey \
            -pubin \
            -outform DER 2>/dev/null
    ) \
    <(
        openssl pkey \
            -in "$TMP/server.key" \
            -pubout \
            -outform DER 2>/dev/null
    )
then
    log "ERROR: certificate and private key do not match"
    exit 1
fi


#
# Back up the current certificate
#
if [[ -e "$CERT" ]]; then
    cp -a "$CERT" "$TMP/old.crt"
fi

if [[ -e "$KEY" ]]; then
    cp -a "$KEY" "$TMP/old.key"
fi


rollback() {
    log "rolling back certificate"

    if [[ -e "$TMP/old.crt" ]]; then
        cp -a "$TMP/old.crt" "$CERT"
    else
        rm -f "$CERT"
    fi

    if [[ -e "$TMP/old.key" ]]; then
        cp -a "$TMP/old.key" "$KEY"
    else
        rm -f "$KEY"
    fi
}


#
# Prepare the files for nginx, then rename them.
#
# nginx itself continues holding the old certificate until it is
# reloaded/restarted, so even if the crt/key files briefly do not match
# between the two renames, this does not affect the running nginx process.
#
install \
    -o root \
    -g kvmd-nginx \
    -m 0644 \
    "$TMP/server.crt" \
    "${CERT}.new"

install \
    -o root \
    -g kvmd-nginx \
    -m 0640 \
    "$TMP/server.key" \
    "${KEY}.new"

mv -f "${KEY}.new" "$KEY"
mv -f "${CERT}.new" "$CERT"


#
# Validate using the actual nginx configuration generated by PiKVM
#
if ! nginx -t -c /run/kvmd/nginx.conf; then
    log "ERROR: nginx configuration test failed"
    rollback
    exit 1
fi


#
# Restart according to the official PiKVM documentation.
#
if ! systemctl restart kvmd-nginx; then
    log "ERROR: kvmd-nginx restart failed"

    rollback

    # Attempt recovery after restoring the old certificate
    nginx -t -c /run/kvmd/nginx.conf || true
    systemctl restart kvmd-nginx || true

    exit 1
fi


log "certificate successfully installed"

openssl x509 \
    -in "$CERT" \
    -noout \
    -subject \
    -issuer \
    -dates

exit 0
PIKVM_RENEW_EOF
    chmod 0755 /usr/local/libexec/pikvm-tailscale-cert-renew

    cat > /etc/systemd/system/pikvm-tailscale-cert-renew.service <<'PIKVM_SERVICE_EOF'
[Unit]
Description=Renew Tailscale TLS certificate for PiKVM nginx
Wants=network-online.target
After=network-online.target tailscaled.service
Requires=tailscaled.service

[Service]
Type=oneshot
ExecStart=/usr/local/libexec/pikvm-tailscale-cert-renew
TimeoutStartSec=5min
PIKVM_SERVICE_EOF

    cat > /etc/systemd/system/pikvm-tailscale-cert-renew.timer <<'PIKVM_TIMER_EOF'
[Unit]
Description=Periodic Tailscale TLS certificate check for PiKVM

[Timer]
OnBootSec=15min
OnUnitActiveSec=1d
RandomizedDelaySec=30min
AccuracySec=1min
Unit=pikvm-tailscale-cert-renew.service

[Install]
WantedBy=timers.target
PIKVM_TIMER_EOF

    systemctl daemon-reload

    # PiKVM nginx owns HTTPS port 443; Tailscale Serve must be disabled.
    tailscale serve --https=443 off

    # Enable and immediately start the periodic timer.
    systemctl enable --now pikvm-tailscale-cert-renew.timer

    # Run one certificate check/renewal immediately as part of installation.
    systemctl start pikvm-tailscale-cert-renew.service

    ro
    trap - EXIT
)
```

After the block finishes, PiKVM's own nginx will listen on port 443. The timer is already enabled and running because the installer uses `systemctl enable --now`.

The official PiKVM documentation also uses the approach of updating `server.crt/server.key` and running `systemctl restart kvmd-nginx` when installing a Tailscale certificate directly into nginx. ([Pikvm][1])

---

## 5. Starting and Checking Status

```bash
systemctl start pikvm-tailscale-cert-renew.service
```

Check:

```bash
systemctl status pikvm-tailscale-cert-renew.service
```

```bash
journalctl \
    -u pikvm-tailscale-cert-renew.service \
    -n 100 \
    --no-pager
```

Certificate:

```bash
openssl x509 \
    -in /etc/kvmd/nginx/ssl/server.crt \
    -noout \
    -subject \
    -issuer \
    -dates \
    -ext subjectAltName
```

If it succeeds and contains:

```text
DNS:{hostname}.{tsnet}.ts.net
```

then everything is OK.

The timer was already enabled and started by the installation block.

Check:

```bash
systemctl list-timers pikvm-tailscale-cert-renew.timer
```

### Access URL

With this configuration, the certificate name is:

```text
{hostname}.{tsnet}.ts.net
```

so in the browser, always use:

```text
https://{hostname}.{tsnet}.ts.net/
```

For `https://{hostname}/` or `[https://100.x.x.x/](https://100.x.x.x/)`, the connection itself may reach nginx, but the certificate name will not match. Tailscale also explicitly states that HTTPS certificates are for fully qualified `*.ts.net` names, not HTTPS certificates for bare hostnames. ([Tailscale][3])

In other words, this approach **completely eliminates Serve, handles port 443 using only PiKVM’s standard nginx, keeps the filesystem RO during normal operation, and switches it to RW only when the certificate actually needs to be renewed**. It also does not conflict with the automatic generation of `/run/kvmd/nginx.conf`.
