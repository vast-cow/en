---
title: "Systemd and Certbot for Automated TLS Certificate Renewal (with DynamicUser for Privilege Separation)"
description: "Automate certificate issuance under an unprivileged account while reserving deployment for root, with temporary UPnP access, renewal hooks, and validation before service restarts."
pubDatetime: 2026-01-19T09:58:40.606Z
---

## Introduction

This configuration automates the acquisition and renewal of Let's Encrypt TLS certificates using Certbot, and schedules regular execution with systemd services and timers.

The primary security focus is on "running with the least privilege necessary."

Certificate acquisition and UPnP-based port manipulation are performed as a non-privileged user using systemd's `DynamicUser=yes`, while processes requiring root privileges are limited to managing Python packages, deploying certificates, and restarting services.

No fixed, dedicated user is created, and explicit privilege switching using `sudo -u` is also avoided.

Additionally, Python packages are checked for updates before each execution. If `uv` is installed, `uv pip` is prioritized; otherwise, it falls back to `pip`.

## Security Design (Privilege Boundary)

### Non-Privileged Execution with DynamicUser

When `DynamicUser=yes` is specified, systemd assigns a temporary UID/GID during service startup and releases it when the service ends.

This mechanism eliminates the need to manage a fixed, dedicated user.

The following processes are executed with non-privileged access:

- Obtaining the local IP address using `ip` and `jq`
- Adding and removing UPnP port mappings using `upnpc`
- Performing ACME authentication and obtaining/renewing certificates with Certbot
- Creating a flag file to indicate successful renewal

These processes are not granted root privileges.

### Processes Requiring Root

Processes that require root privileges are separated by prefixing the systemd execution command with `+`.

`ExecStartPre=+` updates Certbot's Python virtual environment.

`ExecStartPost=+` copies the obtained certificate to a protected location, and restarts Nginx and StrongSwan.

The normal `ExecStart=` command is executed as a non-privileged user using `DynamicUser=yes`.

The execution order is as follows:

```text
certbot-renew.timer
        |
        v
certbot-renew.service
        |
        +-- ExecStartPre=+update-certbot.sh
        |       root
        |       Python package management
        |
        +-- ExecStart=certbot-renew.sh
        |       DynamicUser
        |       Local IP address retrieval
        |       UPnP port opening
        |       Certbot execution
        |       UPnP port closing
        |
        +-- ExecStartPost=+certbot-deploy.sh
                root
                Renewal flag verification
                Certificate verification and deployment
                Nginx / StrongSwan restart
```

If `ExecStartPre=` or `ExecStart=` fails, `ExecStartPost=` is not executed.

Note that the `+` prefix not only switches the UID/GID but also releases some sandbox restrictions. Therefore, scripts executed with root privileges should be owned by root and not modifiable by non-privileged users.

### Directory Management with Systemd

Persistent and temporary data are managed using the following systemd directives:

- `StateDirectory=`: Certbot configuration, certificates, and private keys
- `LogsDirectory=`: Certbot logs
- `RuntimeDirectory=`: Temporary working area, HTTP-01 challenge, renewal flag

Combining `DynamicUser=yes` and `StateDirectory=` ensures that the persistent directory is protected by systemd.

Therefore, the webroot for the HTTP-01 challenge, which Nginx references from outside, is placed in the protected `RuntimeDirectory` instead of the `StateDirectory`.

### Deploy Only When an Update Occurs

A flag file is created using Certbot's `--deploy-hook`.

The subsequent root process copies the certificate and restarts the service only if a new certificate is issued or an existing one is updated.

Deployment is skipped if no changes are made to the certificate.

## File Structure

The project has the following structure:

```text
certbot-renewal/
├── certbot-renew.service
├── certbot-renew.timer
├── certbot.env.example
├── update-certbot.sh
├── certbot-renew.sh
├── certbot-deploy.sh
├── install.sh
└── uninstall.sh
```

The main locations after installation are as follows:

```text
/etc/
├── certbot-renewal/
│   └── certbot.env
└── systemd/system/
    ├── certbot-renew.service
    └── certbot-renew.timer

/usr/libexec/
└── certbot-renewal/
    ├── update-certbot.sh
    ├── certbot-renew.sh
    ├── certbot-deploy.sh
    └── venv/

/var/lib/
└── certbot-renewal/
    └── config/

/var/log/
└── certbot-renewal/

/run/
└── certbot-renewal/
    ├── work/
    ├── webroot/
    └── renewed
```

`/var/lib/certbot-renewal` and `/var/log/certbot-renewal` may actually be managed through systemd's private directories.

## certbot-renew.service

```ini
[Unit]
Description=Renew and deploy TLS certificates using Certbot
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot

EnvironmentFile=/etc/certbot-renewal/certbot.env

DynamicUser=yes

StateDirectory=certbot-renewal
StateDirectoryMode=0700

RuntimeDirectory=certbot-renewal
RuntimeDirectoryMode=0755

LogsDirectory=certbot-renewal
LogsDirectoryMode=0700

PrivateTmp=yes
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes

# HTTP-01 challenge files must be readable by Nginx.
UMask=0022

ExecStartPre=+/usr/libexec/certbot-renewal/update-certbot.sh
ExecStart=/usr/libexec/certbot-renewal/certbot-renew.sh
ExecStartPost=+/usr/libexec/certbot-renewal/certbot-deploy.sh
```

`Type=oneshot` causes each phase to be executed sequentially.

`UMask=0022` is a setting to allow Nginx to read the HTTP-01 challenge files created by Certbot during non-privileged execution.

On the other hand, persistent data such as private keys is protected by `StateDirectoryMode=0700`. Certbot itself also sets restricted access permissions for private keys.

`NoNewPrivileges=yes` and `ProtectSystem=strict` restrict normal non-privileged processes.

## certbot-renew.timer

```ini
[Unit]
Description=Periodically renew TLS certificates using Certbot

[Timer]
Unit=certbot-renew.service

OnCalendar=*-*-* 00:00:00
OnCalendar=*-*-* 12:00:00

RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

Certbot is executed twice a day.

`RandomizedDelaySec=1h` distributes the execution time, and `Persistent=true` compensates for missed executions after a shutdown.

If the certificate has sufficient remaining validity, Certbot will exit without renewing.

## update-certbot.sh

This script is executed as root and creates or updates Certbot's dedicated Python virtual environment.

It checks for package updates on each execution, so it is executed every time the timer starts.

If `uv` is available, `uv pip` is prioritized.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

VENV="$CERTBOT_VENV"

echo "[info] preparing Certbot virtual environment" >&2

if [[ ! -x "$VENV/bin/python" ]]; then
    echo "[info] creating virtual environment" >&2

    # The virtual environment must remain root-owned.
    umask 022
    python3 -m venv "$VENV"
fi

echo "[info] updating Python packages" >&2

if command -v uv >/dev/null 2>&1; then
    echo "[info] using uv pip" >&2

    uv pip install \
        --python "$VENV/bin/python" \
        --upgrade \
        certbot
else
    echo "[info] using pip" >&2

    if ! "$VENV/bin/python" -m pip --version \
        >/dev/null 2>&1; then
        "$VENV/bin/python" -m ensurepip --upgrade
    fi

    "$VENV/bin/python" -m pip install \
        --upgrade pip

    "$VENV/bin/python" -m pip install \
        --upgrade \
        --upgrade-strategy eager \
        certbot
fi

echo "[info] checking installed dependencies" >&2

if command -v uv >/dev/null 2>&1; then
    uv pip check --python "$VENV/bin/python"
else
    "$VENV/bin/python" -m pip check
fi

echo "[info] installed Certbot version:" >&2
"$VENV/bin/certbot" --version

echo "[info] Python package update completed" >&2
```

### Using uv and pip

`uv pip install --python` allows you to explicitly specify the target Python virtual environment.

`--upgrade` resolves and installs the newest compatible version of Certbot and its required dependencies.

With regular pip, `--upgrade-strategy eager` is also specified to actively update dependencies.

In either case, it does not update arbitrary Python packages that are not included in Certbot's dependencies. The update target is Certbot and its dependencies.

Because this configuration attempts to update packages every time, the certificate renewal process will also fail if access to the Python package source is unavailable or if dependency resolution fails. Additionally, if the latest release of a package contains bugs, it may affect certificate renewal.

In a production environment, consider these risks and consider error monitoring, backups before updates, and version fixing as needed.

## certbot-renew.sh

This script executes Certbot and UPnP in a non-privileged user.

No root checks or explicit privilege switching using `sudo -u` is necessary.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID == 0 )); then
    echo "[error] must run as non-root" >&2
    exit 1
fi

FLAGFILE="/run/certbot-renewal/renewed"

# Prepare writable runtime directories.
install -d -m 0700 "$WORK_DIR"
install -d -m 0755 "$WEB_ROOT"

echo "[info] detecting local IP address" >&2

local_ip="$(
    ip -j route get 1.1.1.1 |
        jq -r '.[0].prefsrc // empty'
)"

if [[ -z "$local_ip" ]]; then
    echo "[error] failed to detect local IP" >&2
    exit 1
fi

echo "[info] local IP: $local_ip" >&2

mapping_attempted=0

cleanup() {
    if (( mapping_attempted )); then
        echo "[info] closing UPnP port" >&2
        upnpc -d 80 tcp || true
    fi
}

trap cleanup EXIT
trap 'exit 143' TERM
trap 'exit 130' INT

echo "[info] opening UPnP port" >&2

# WAN:80 -> LAN:80, lease: 600 seconds
mapping_attempted=1
upnpc -a "$local_ip" 80 80 tcp 600

echo "[info] running Certbot" >&2

"$CERTBOT_VENV/bin/certbot" \
    certonly \
    --non-interactive \
    --quiet \
    --webroot \
    -w "$WEB_ROOT" \
    --cert-name "$CERT_NAME" \
    --key-type rsa \
    --rsa-key-size 4096 \
    -d "$DOMAINLIST" \
    --agree-tos \
    --no-eff-email \
    --email "$EMAIL" \
    --deploy-hook "/usr/bin/touch $FLAGFILE" \
    --config-dir "$CONFIG_PATH" \
    --work-dir "$WORK_DIR" \
    --logs-dir "$LOGS_DIR"

echo "[info] Certbot completed successfully" >&2
```

## Handling UPnP Port Mappings

A mapping from WAN:80 to LAN:80 is temporarily added for HTTP-01 validation.

The mapping is removed by the `trap` upon Certbot's completion.

However, cleanup may not be executed in cases such as SIGKILL, power failure, or abnormal process termination.

Therefore, if the router supports it, the lease time is set to 600 seconds to prevent the mapping from remaining indefinitely.

Also, `upnpc -d 80 tcp` is an operation to delete the mapping for the specified port. It is not suitable for configurations that share the same port for other purposes.

## Renewal Flag

A flag file is created using Certbot's `--deploy-hook`.

A certificate is copied and the service is restarted by the subsequent root process only if a new certificate is issued or an existing one is updated.

Deployment is skipped if the certificate does not change.

## File Structure

The project has the following structure:

```text
certbot-renewal/
├── certbot-renew.service
├── certbot-renew.timer
├── certbot.env.example
├── update-certbot.sh
├── certbot-renew.sh
├── certbot-deploy.sh
├── install.sh
└── uninstall.sh
```

The main locations after installation are as follows:

```text
/etc/
├── certbot-renewal/
│   └── certbot.env
└── systemd/system/
    ├── certbot-renew.service
    └── certbot-renew.timer

/usr/libexec/
└── certbot-renewal/
    ├── update-certbot.sh
    ├── certbot-renew.sh
    ├── certbot-deploy.sh
    └── venv/

/var/lib/
└── certbot-renewal/
    └── config/

/var/log/
└── certbot-renewal/

/run/
└── certbot-renewal/
    ├── work/
    ├── webroot/
    └── renewed
```

`/var/lib/certbot-renewal` and `/var/log/certbot-renewal` may actually be managed through systemd's private directories.

## certbot-renew.service

```ini
[Unit]
Description=Renew and deploy TLS certificates using Certbot
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot

EnvironmentFile=/etc/certbot-renewal/certbot.env

DynamicUser=yes

StateDirectory=certbot-renewal
StateDirectoryMode=0700

RuntimeDirectory=certbot-renewal
RuntimeDirectoryMode=0755

LogsDirectory=certbot-renewal
LogsDirectoryMode=0700

PrivateTmp=yes
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes

# HTTP-01 challenge files must be readable by Nginx.
UMask=0022

ExecStartPre=+/usr/libexec/certbot-renewal/update-certbot.sh
ExecStart=/usr/libexec/certbot-renewal/certbot-renew.sh
ExecStartPost=+/usr/libexec/certbot-renewal/certbot-deploy.sh
```

`Type=oneshot` causes each phase to be executed sequentially.

`UMask=0022` is a setting to allow Nginx to read the HTTP-01 challenge files created by Certbot during non-privileged execution.

On the other hand, persistent data such as private keys is protected by `StateDirectoryMode=0700`. Certbot itself also sets restricted access permissions for private keys.

`NoNewPrivileges=yes` and `ProtectSystem=strict` restrict normal non-privileged processes.

## certbot-renew.timer

```ini
[Unit]
Description=Periodically renew TLS certificates using Certbot

[Timer]
Unit=certbot-renew.service

OnCalendar=*-*-* 00:00:00
OnCalendar=*-*-* 12:00:00

RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

Certbot is executed twice a day.

`RandomizedDelaySec=1h` distributes the execution time, and `Persistent=true` compensates for missed executions after a shutdown.

If the certificate has sufficient remaining validity, Certbot will exit without renewing.

## update-certbot.sh

This script is executed as root and creates or updates Certbot's dedicated Python virtual environment.

It checks for package updates on each execution, so it is executed every time the timer starts.

If `uv` is available, `uv pip` is prioritized.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

VENV="$CERTBOT_VENV"

echo "[info] preparing Certbot virtual environment" >&2

if [[ ! -x "$VENV/bin/python" ]]; then
    echo "[info] creating virtual environment" >&2

    # The virtual environment must remain root-owned.
    umask 022
    python3 -m venv "$VENV"
fi

echo "[info] updating Python packages" >&2

if command -v uv >/dev/null 2>&1; then
    echo "[info] using uv pip" >&2

    uv pip install \
        --python "$VENV/bin/python" \
        --upgrade \
        certbot
else
    echo "[info] using pip" >&2

    if ! "$VENV/bin/python" -m pip --version \
        >/dev/null 2>&1; then
        "$VENV/bin/python" -m ensurepip --upgrade
    fi

    "$VENV/bin/python" -m pip install \
        --upgrade pip

    "$VENV/bin/python" -m pip install \
        --upgrade \
        --upgrade-strategy eager \
        certbot
fi

echo "[info] checking installed dependencies" >&2

if command -v uv >/dev/null 2>&1; then
    uv pip check --python "$VENV/bin/python"
else
    "$VENV/bin/python" -m pip check
fi

echo "[info] installed Certbot version:" >&2
"$VENV/bin/certbot" --version

echo "[info] Python package update completed" >&2
```

## Using uv and pip

`uv pip install --python` allows you to explicitly specify the target Python virtual environment.

`--upgrade` resolves and installs the newest compatible version of Certbot and its required dependencies.

With regular pip, `--upgrade-strategy eager` is also specified to actively update dependencies.

In either case, it does not update arbitrary Python packages that are not included in Certbot's dependencies. The update target is Certbot and its dependencies.

Because this configuration attempts to update packages every time, the certificate renewal process will also fail if access to the Python package source is unavailable or if dependency resolution fails. Additionally, if the latest release of a package contains bugs, it may affect certificate renewal.

In a production environment, consider these risks and consider error monitoring, backups before updates, and version fixing as needed.

## certbot-renew.sh

This script executes Certbot and UPnP in a non-privileged user.

No root checks or explicit privilege switching using `sudo -u` is necessary.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID == 0 )); then
    echo "[error] must run as non-root" >&2
    exit 1
fi

FLAGFILE="/run/certbot-renewal/renewed"

# Prepare writable runtime directories.
install -d -m 0700 "$WORK_DIR"
install -d -m 0755 "$WEB_ROOT"

echo "[info] detecting local IP address" >&2

local_ip="$(
    ip -j route get 1.1.1.1 |
        jq -r '.[0].prefsrc // empty'
)"

if [[ -z "$local_ip" ]]; then
    echo "[error] failed to detect local IP" >&2
    exit 1
fi

echo "[info] local IP: $local_ip" >&2

mapping_attempted=0

cleanup() {
    if (( mapping_attempted )); then
        echo "[info] closing UPnP port" >&2
        upnpc -d 80 tcp || true
    fi
}

trap cleanup EXIT
trap 'exit 143' TERM
trap 'exit 130' INT

echo "[info] opening UPnP port" >&2

# WAN:80 -> LAN:80, lease: 600 seconds
mapping_attempted=1
upnpc -a "$local_ip" 80 80 tcp 600

echo "[info] running Certbot" >&2

"$CERTBOT_VENV/bin/certbot" \
    certonly \
    --non-interactive \
    --quiet \
    --webroot \
    -w "$WEB_ROOT" \
    --cert-name "$CERT_NAME" \
    --key-type rsa \
    --rsa-key-size 4096 \
    -d "$DOMAINLIST" \
    --agree-tos \
    --no-eff-email \
    --email "$EMAIL" \
    --deploy-hook "/usr/bin/touch $FLAGFILE" \
    --config-dir "$CONFIG_PATH" \
    --work-dir "$WORK_DIR" \
    --logs-dir "$LOGS_DIR"

echo "[info] Certbot completed successfully" >&2
```

## Handling UPnP Port Mappings

A mapping from WAN:80 to LAN:80 is temporarily added for HTTP-01 validation.

The mapping is removed by the `trap` upon Certbot's completion.

However, cleanup may not be executed in cases such as SIGKILL, power failure, or abnormal process termination.

Therefore, if the router supports it, the lease time is set to 600 seconds to prevent the mapping from remaining indefinitely.

Also, `upnpc -d 80 tcp` is an operation to delete the mapping for the specified port. It is not suitable for configurations that share the same port for other purposes.

## Renewal Flag

A flag file is created using Certbot's `--deploy-hook`.

A certificate is copied and the service is restarted by the subsequent root process only if a new certificate is issued or an existing one is updated.

Deployment is skipped if the certificate does not change.

## certbot-deploy.sh

This script, executed as root, deploys the updated certificate to Nginx and StrongSwan.

If the certificate has not changed, it does nothing and exits.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

FLAGFILE="/run/certbot-renewal/renewed"

if [[ ! -f "$FLAGFILE" ]]; then
    echo "[info] no certificate renewal; skip deploy" >&2
    exit 0
fi

echo "[info] deploying renewed certificates" >&2

CERTPATH="$CONFIG_PATH/live/$CERT_NAME"

CONFIG_REAL="$(realpath -e -- "$CONFIG_PATH")"
ARCHIVE_PREFIX="$CONFIG_REAL/archive/$CERT_NAME/"

check_cert_file() {
    local name="$1"
    local source

    source="$(realpath -e -- "$CERTPATH/$name")" \
        || return 1

    [[ "$source" == "$ARCHIVE_PREFIX"* ]] \
        || return 1

    [[ -f "$source" && -s "$source" ]] \
        || return 1

    printf '%s\n' "$source"
}

FULLCHAIN="$(check_cert_file fullchain.pem)" || exit 2
PRIVKEY="$(check_cert_file privkey.pem)" || exit 2
CHAIN="$(check_cert_file chain.pem)" || exit 2

echo "[info] preparing destination directories" >&2

install -d -m 0755 "$NGINX_CERTS"
install -d -m 0755 "$IPSEC_ETC/private"
install -d -m 0755 "$IPSEC_ETC/cacerts"
install -d -m 0755 "$IPSEC_ETC/certs"

echo "[info] installing Nginx certificates" >&2

install -m 0644 \
    "$FULLCHAIN" \
    "$NGINX_CERTS/fullchain.pem"

install -m 0600 \
    "$PRIVKEY" \
    "$NGINX_CERTS/privkey.pem"

echo "[info] installing StrongSwan certificates" >&2

install -m 0600 \
    "$PRIVKEY" \
    "$IPSEC_ETC/private/privkey.pem"

install -m 0644 \
    "$CHAIN" \
    "$IPSEC_ETC/cacerts/chain.pem"

install -m 0644 \
    "$FULLCHAIN" \
    "$IPSEC_ETC/certs/fullchain.pem"

echo "[info] validating Nginx configuration" >&2

nginx -t

echo "[info] restarting Nginx" >&2
systemctl restart nginx

echo "[info] restarting StrongSwan" >&2
systemctl restart strongswan-starter

echo "[info] certificate deployment completed" >&2
```

Before deploying the certificate, it checks that the necessary files exist and are not empty.

It also checks that the symbolic links in `live/` point to the expected `archive/` location.

However, this file check is a limited safeguard. Because the certificate storage area managed by a non-privileged user is read by root, it does not provide complete protection against malicious simultaneous modifications or link substitutions.

In a strict threat model, you should add integrity checks for certificates and private keys, secure reading using file descriptors, and atomic deployment after verifying service configurations.

Also, this example does not have a complete rollback mechanism for failed deployments. If availability is important in production, consider using a staging directory and atomic replacement.

## certbot.env.example

```ini
CERT_NAME=hoge.ddns.net
DOMAINLIST=hoge.ddns.net
EMAIL=hoge@mail.com

CERTBOT_VENV=/usr/libexec/certbot-renewal/venv

CONFIG_PATH=/var/lib/certbot-renewal/config
WORK_DIR=/run/certbot-renewal/work
LOGS_DIR=/var/log/certbot-renewal
WEB_ROOT=/run/certbot-renewal/webroot

NGINX_CERTS=/etc/nginx/pki/certs
IPSEC_ETC=/etc/ipsec.d
```

For directories managed by systemd, you do not need to manually set the owner during installation.

`DOMAINLIST` assumes a single domain in this case.

If you specify multiple domains, extend the script to pass multiple `-d` options to Certbot.

## install.sh

The installation script checks for the presence of commands, places files, initializes the virtual environment, and enables the systemd timer.

The Python package update process is consolidated in `update-certbot.sh`, so the same script is used during installation.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

ETC_DIR=/etc/certbot-renewal
LIBEXEC_DIR=/usr/libexec/certbot-renewal
SYSTEMD_DIR=/etc/systemd/system

missing=()

for cmd in ip upnpc jq python3 systemctl nginx; do
    if ! command -v "$cmd" >/dev/null 2>&1; then
        missing+=("$cmd")
    fi
done

if ! ip -j address >/dev/null 2>&1; then
    missing+=("iproute2 JSON support")
fi

if (( ${#missing[@]} > 0 )); then
    echo "[error] missing requirements:" >&2

    for item in "${missing[@]}"; do
        echo "  - $item" >&2
    done

    exit 1
fi

echo "[info] creating installation directories" >&2

install -d -m 0700 "$ETC_DIR"
install -d -m 0755 "$LIBEXEC_DIR"

echo "[info] installing configuration" >&2

if [[ ! -e "$ETC_DIR/certbot.env" ]]; then
    install -m 0600 \
        certbot.env.example \
        "$ETC_DIR/certbot.env"
else
    echo "[info] preserving existing configuration" >&2
fi

echo "[info] installing scripts" >&2

install -m 0755 \
    update-certbot.sh \
    certbot-renew.sh \
    certbot-deploy.sh \
    "$LIBEXEC_DIR/"

echo "[info] installing systemd units" >&2

install -m 0644 \
    certbot-renew.service \
    certbot-renew.timer \
    "$SYSTEMD_DIR/"

echo "[info] preparing Python virtual environment" >&2

# Import configuration for the initial package installation.
set -a
source "$ETC_DIR/certbot.env"
set +a

"$LIBEXEC_DIR/update-certbot.sh"

echo "[info] reloading systemd" >&2

systemctl daemon-reload

echo "[info] enabling renewal timer" >&2

systemctl enable --now certbot-renew.timer

echo "[info] installation completed" >&2
echo "[info] edit $ETC_DIR/certbot.env if needed" >&2
```

## Notes for Installation

If the environment configuration file already exists, `install.sh` will not overwrite it.

However, sample values are placed during the initial installation, so you need to check the configuration before running it.

Python's `venv` feature must be available. On Debian-based distributions, you may need to install the `python3-venv` package separately.

`uv` is an optional dependency, and `pip` can be used even if it is not installed.

## uninstall.sh

Uninstallation stops the systemd timer and service, removes related scripts, the virtual environment, and unit files.

To avoid accidentally losing certificates or private keys, the normal uninstallation does not delete Certbot's persistent data and configuration files.

Only when `--purge` is specified, the configuration, certificates, logs, etc. are deleted.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

ETC_DIR=/etc/certbot-renewal
LIBEXEC_DIR=/usr/libexec/certbot-renewal
SYSTEMD_DIR=/etc/systemd/system

PURGE=false

case "${1:-}" in
    "")
        ;;
    --purge)
        PURGE=true
        ;;
    *)
        echo "Usage: $0 [--purge]" >&2
        exit 2
        ;;
esac

echo "[info] stopping renewal timer" >&2

systemctl disable --now certbot-renew.timer \
    2>/dev/null || true

echo "[info] stopping renewal service" >&2

systemctl stop certbot-renew.service \
    2>/dev/null || true

echo "[info] removing systemd units" >&2

rm -f -- \
    "$SYSTEMD_DIR/certbot-renew.service" \
    "$SYSTEMD_DIR/certbot-renew.timer"

systemctl daemon-reload
systemctl reset-failed certbot-renew.service \
    2>/dev/null || true

echo "[info] removing application scripts and venv" >&2

rm -rf -- "$LIBEXEC_DIR"

if [[ "$PURGE" == true ]]; then
    echo "[warn] purging Certbot configuration and certificates" >&2

    rm -rf -- "$ETC_DIR"

    # Remove both public paths and DynamicUser private backing data.
    rm -rf -- \
        /var/lib/certbot-renewal \
        /var/lib/private/certbot-renewal \
        /var/log/certbot-renewal \
        /var/log/private/certbot-renewal

    echo "[info] purge completed" >&2
else
    echo "[info] keeping configuration and certificates" >&2
    echo "[info] configuration: $ETC_DIR" >&2
    echo "[info] state: /var/lib/certbot-renewal" >&2
    echo "[info] logs: /var/log/certbot-renewal" >&2
fi

echo "[info] uninstall completed" >&2
```

## Uninstallation Behavior

Normal uninstallation:

```bash
sudo ./uninstall.sh
```

Removes the following:

- systemd timer and service unit files
- Installed scripts
- Certbot Python virtual environment

Keeps the certificate, private key, Certbot update history, and configuration files.

Complete removal:

```bash
sudo ./uninstall.sh --purge
```

In addition to the normal removal, removes Certbot configuration files, certificates, private keys, and logs.

**Caution:** `--purge` will also remove the account information and private keys needed to re-acquire the certificate. Back up any necessary data before running.

Certificate files already deployed to Nginx are not deleted. These may be referenced from other configurations, so they must be checked and deleted separately.

Changes to the Nginx configuration are also not automatically reverted.

## Nginx HTTP-01 Configuration

Because the Certbot webroot is changed to `/run/certbot-renewal/webroot`, Nginx must also reference the same directory.

```nginx
server {
    listen 80;
    server_name hoge.ddns.net;

    location ^~ /.well-known/acme-challenge/ {
        root /run/certbot-renewal/webroot;
        default_type text/plain;
        try_files $uri =404;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}
```

`RuntimeDirectory=certbot-renewal` means that `/run/certbot-renewal` exists during service startup.

Also, `RuntimeDirectoryMode=0755` and the webroot `0755` allow Nginx to read the HTTP-01 challenge files.

If Nginx itself has its own sandbox restrictions, such as `ProtectSystem=` or `InaccessiblePaths=`, make sure that Nginx can access the webroot.

HTTP-01 validation requires that the TCP/80 port of the domain be accessible from the outside.

## Installation Method

First, create the necessary files and edit the configuration.

```bash
cp certbot.env.example certbot.env
```

Edit the contents of `certbot.env` according to your environment and install.

```bash
sudo ./install.sh
```

Check the installed configuration file.

```bash
sudoedit /etc/certbot-renewal/certbot.env
```

Run manually for the first time.

```bash
sudo systemctl start certbot-renew.service
```

Check the operation log.

```bash
sudo journalctl \
    -u certbot-renew.service \
    -n 100 \
    --no-pager
```

Check the status of the timer.

```bash
systemctl list-timers certbot-renew.timer
```

Use the following to check the systemd unit configuration:

```bash
sudo systemd-analyze verify \
    /etc/systemd/system/certbot-renew.service
```

Use the following to evaluate permission settings:

```bash
systemd-analyze security certbot-renew.service
```

However, `systemd-analyze security` is a static configuration evaluation and does not guarantee actual attack resistance.

## Migration from Existing Environment

If migrating from an old configuration that used a fixed user, you need to move Certbot's persistent data to the new `StateDirectory`.

First, stop the old timer/service and back up the certificates, private keys, ACME account information, and renewal settings.

Simply copying the contents of the old directory may not allow the user assigned by `DynamicUser=yes` to access it correctly.

Migrate, considering systemd's private directory management and ownership processing, and after verifying that it is working, delete the old configuration.

In particular, the symbolic links from `live/` to `archive/` may be broken by the copy method.

## Operational Notes

### Package Updates Every Time

This configuration checks for Python package updates each time the certificate is renewed.

If a new package introduces compatibility issues, certificate renewal may fail.

In continuous operation, it is recommended to monitor systemd's failure logs and be able to quickly detect problems.

Also, the package update, which requires communication, is performed by root privileges separately from the non-privileged ACME communication. The trustworthiness of the package source and the write permissions of the virtual environment must be strictly managed.

### Certificate Validity Monitoring

Even if the systemd timer is running correctly, certificate renewal may fail due to problems with DNS, routers, UPnP, or Let's Encrypt.

It is recommended to monitor not only the operation of the timer but also the actual certificate validity.

### Deployment Destination Management

The Nginx and StrongSwan certificate directories are managed by root and are not given write permissions to non-privileged users.

If the number of deployment destinations increases, the non-privileged Certbot process should not be changed, and the processing should be added to the root deployment script.

### Dependencies on External Services

Certificate acquisition depends on Let's Encrypt's ACME service, Python package updates depend on package indexes, and UPnP operations depend on the router's implementation.

It is not possible to completely eliminate these failures, so proper logging and failure monitoring are necessary.

## Summary

In this configuration, systemd's `DynamicUser=yes` is used to execute the certificate renewal process as a non-privileged user, and processes that require root privileges are separated by prefixing the systemd execution command with `+`.

This eliminates the need to create a fixed user and consolidates ownership and runtime directory management with systemd.

Furthermore, Certbot's Python virtual environment is managed by root, and Python packages are checked for updates each time the script is executed.

`install.sh` and `uninstall.sh` are used to manage installation and removal, and normal uninstallation retains the certificate data.

The key to this configuration is to clearly separate the different responsibilities of certificate acquisition, Python package management, and deployment to the production environment, and to grant only the permissions necessary for each process.
