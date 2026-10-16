---
title: "Systemd and Certbot for Automated TLS Certificate Renewal (with DynamicUser for Privilege Separation)"
description: "Automate certificate issuance under an unprivileged account while reserving deployment for root, with temporary UPnP access, renewal hooks, and validation before service restarts."
pubDatetime: 2026-01-19T09:58:40.606Z
---

## Introduction

This configuration automates the acquisition and renewal of Let's Encrypt TLS certificates using Certbot and executing them periodically with systemd services and timers, while separating permissions using DynamicUser.

The main security focus is on "running with the least privileges necessary."

Certificate acquisition and UPnP port operations are performed as a non-privileged user using systemd's `DynamicUser=yes`, and operations requiring root privileges are limited to managing Python packages, deploying certificates, and restarting services.

A fixed dedicated user is not created, and explicit privilege switching using `sudo -u` is not used.

Furthermore, Python packages are checked for updates before each execution. If `uv` is installed, `uv pip` is prioritized, and if not, `pip` is used as a fallback.

## Security Design (Permission Boundaries)

### Non-Privileged Execution with DynamicUser

When `DynamicUser=yes` is specified, systemd assigns a temporary UID/GID at service startup and releases it at service termination.

This eliminates the need to manage a fixed dedicated user.

The following processes are executed with non-privileged access:

- Obtaining the local IP address using `ip` and `jq`.
- Adding or removing UPnP port mappings using `upnpc`.
- Performing ACME authentication and obtaining/renewing certificates using Certbot.
- Creating a flag file to indicate successful renewal.

These processes are not granted root privileges.

### Processes Using Root

Processes requiring root privileges are separated by attaching the `+` prefix to the systemd execution command.

`ExecStartPre=+` updates Certbot's Python virtual environment.

`ExecStartPost=+` copies the obtained certificate to a protected location and restarts Nginx and StrongSwan.

The regular `ExecStart=` is executed as a non-privileged user using `DynamicUser=yes`.

The execution order is as follows:

```text
certbot-renew.timer
        |
        v
certbot-renew.service
        |
        +-- ExecStartPre=+update-certbot.sh
        |       root
        |       Python package update
        |
        +-- ExecStart=certbot-renew.sh
        |       DynamicUser
        |       Local IP acquisition
        |       UPnP port opening
        |       Certbot execution
        |       UPnP port closing
        |
        +-- ExecStartPost=+certbot-deploy.sh
                root
                Update flag check
                Certificate verification and deployment
                Nginx / StrongSwan restart
```

If `ExecStartPre=` or `ExecStart=` fails, `ExecStartPost=` is not executed.

Note that the `+` prefix not only switches UID/GID but also releases some sandbox restrictions. Therefore, scripts executed as root should be owned by root and not modifiable by non-privileged users.

### Directory Management with systemd

Persistent data and temporary data are managed using the following systemd directives:

- `StateDirectory=`: Certbot configuration, certificates, and private keys
- `LogsDirectory=`: Certbot logs
- `RuntimeDirectory=`: Temporary working area, HTTP-01 challenge, renewal flag

Combining `DynamicUser=yes` and `StateDirectory=` protects the persistent directory with systemd.

Therefore, the webroot for HTTP-01 challenges, which Nginx references externally, is placed in the protected `StateDirectory` rather than in the `RuntimeDirectory`.

### Deploy Only When an Update Occurs

A flag file is created with Certbot's `--deploy-hook`.

Only if the certificate is newly issued or updated, the subsequent root process copies the certificate and restarts the service.

If there are no changes to the certificate, the deployment is skipped.

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

`/var/lib/certbot-renewal` and `/var/log/certbot-renewal` may be managed via systemd's private directories in practice.

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

`Type=oneshot` causes each phase to be executed in order.

`UMask=0022` is a setting to ensure that the HTTP-01 challenge file created by Certbot, which is executed as a non-privileged user, can be read by Nginx.

On the other hand, persistent data such as private keys is protected with `StateDirectoryMode=0700`. Certbot itself also sets restricted access permissions for private keys.

`NoNewPrivileges=yes` and `ProtectSystem=strict` restrict the normal non-privileged process.

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

`RandomizedDelaySec=1h` distributes the execution time, and `Persistent=true` compensates for missed executions during shutdown.

If there is sufficient margin in the certificate validity period, Certbot will exit without updating.

## update-certbot.sh

This script is executed as root and creates or updates Certbot's dedicated Python virtual environment.

It checks for package updates before each execution, so it is executed every time the timer is started.

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

By using `uv pip install --python`, the target Python virtual environment can be explicitly specified.

`--upgrade` resolves the new compatible version of Certbot and the necessary dependent packages.

With regular pip, `--upgrade-strategy eager` is also specified to actively update dependent packages.

In either case, only Certbot and its dependencies are updated, and arbitrary Python packages not included in Certbot's dependencies are not updated.

Since this configuration attempts to update packages every time, the certificate renewal process will fail if the Python package repository is inaccessible or if dependency resolution fails. In addition, the latest release of the package may have bugs, which may affect certificate renewal.

In a production environment, consider the risks and implement error monitoring, pre-update backups, and version locking as needed.

## certbot-renew.sh

This script executes Certbot and UPnP as a non-privileged user.

No root checks or privilege switching using `sudo -u` is required.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID == 0 )); then
    echo "[error] must run as non-root" >&2
    exit 1
fi

# Paths default to directories managed by systemd.
CONFIG_PATH="${CONFIG_PATH:-${STATE_DIRECTORY:?STATE_DIRECTORY is required}/config}"
WORK_DIR="${WORK_DIR:-${RUNTIME_DIRECTORY:?RUNTIME_DIRECTORY is required}/work}"
LOGS_DIR="${LOGS_DIR:-${LOGS_DIRECTORY:?LOGS_DIRECTORY is required}}"
WEB_ROOT="${WEB_ROOT:-${RUNTIME_DIRECTORY:?RUNTIME_DIRECTORY is required}/webroot}"
FLAGFILE="${RUNTIME_DIRECTORY:?RUNTIME_DIRECTORY is required}/renewed"

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

### Handling UPnP Port Mapping

A port mapping from WAN:80 to LAN:80 is temporarily added for the HTTP-01 verification.

The mapping is deleted by `trap` upon Certbot's termination.

However, if there is a SIGKILL, power outage, or abnormal process termination, the cleanup may not be executed.

Therefore, if the router supports it, the lease time is set to 600 seconds to prevent the mapping from remaining indefinitely.

Also, `upnpc -d 80 tcp` is an operation to delete the port mapping for the specified port. This is not suitable for configurations that share the same port mapping for other purposes.

### Update Flag

A flag file is created with Certbot's `--deploy-hook` when the certificate is issued or updated.

The subsequent root process only copies the certificate and restarts the service if the certificate has been newly issued or updated.

If there are no changes to the certificate, the deployment is skipped.

## certbot-deploy.sh

This script is for root only and deploys the updated certificate to Nginx and StrongSwan.

If the certificate has not changed, it does nothing and exits.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

if (( EUID != 0 )); then
    echo "[error] must run as root" >&2
    exit 1
fi

CONFIG_PATH="${CONFIG_PATH:-${STATE_DIRECTORY:?STATE_DIRECTORY is required}/config}"
FLAGFILE="${RUNTIME_DIRECTORY:?RUNTIME_DIRECTORY is required}/renewed"

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

It also checks that the symbolic link from `live/` to `archive/` points to the expected location.

However, this file check is a limited security measure. Since the certificate storage area managed by a non-privileged user is read by root, it does not completely protect against malicious simultaneous changes or link substitutions.

In a strict threat model, additional measures are needed, such as handling the certificate storage area, secure reading using file descriptors, certificate integrity verification, and atomic deployment after service configuration verification.

Also, this example does not have a complete rollback function for failed copies. If availability is important in a production environment, consider using a staging directory and atomic replacement.

## certbot.env.example

```ini
CERT_NAME=hoge.ddns.net
DOMAINLIST=hoge.ddns.net
EMAIL=hoge@mail.com

CERTBOT_VENV=/usr/libexec/certbot-renewal/venv

# Optional overrides (uncomment only when the defaults are unsuitable):
# CONFIG_PATH=/custom/certbot/config
# WORK_DIR=/custom/certbot/work
# LOGS_DIR=/custom/certbot/logs
# WEB_ROOT=/custom/certbot/webroot

NGINX_CERTS=/etc/nginx/pki/certs
IPSEC_ETC=/etc/ipsec.d
```

For directories managed by systemd, you do not need to manually set the owner during installation.

In `certbot.env`, if there is no explicit specification, the path is derived from the systemd environment variable.

| Setting | Default Value | Reference |
| --- | --- | --- |
| `CONFIG_PATH` | `$STATE_DIRECTORY/config` | `StateDirectory=` |
| `WORK_DIR` | `$RUNTIME_DIRECTORY/work` | `RuntimeDirectory=` |
| `LOGS_DIR` | `$LOGS_DIRECTORY` | `LogsDirectory=` |
| `WEB_ROOT` | `$RUNTIME_DIRECTORY/webroot` | `RuntimeDirectory=` |
| `FLAGFILE` (internal variable) | `$RUNTIME_DIRECTORY/renewed` | `RuntimeDirectory=` |

`FLAGFILE` is not specified in the configuration file and is always assembled from `RUNTIME_DIRECTORY`. `CERTBOT_VENV`, `NGINX_CERTS`, and `IPSEC_ETC` cannot be derived from the systemd-provided environment variables, so explicit settings are maintained.

Systemd provides the environment variables `RUNTIME_DIRECTORY`, `STATE_DIRECTORY`, `CACHE_DIRECTORY`, `LOGS_DIRECTORY`, and `CONFIGURATION_DIRECTORY`. This configuration does not use `CacheDirectory=` and `ConfigurationDirectory=`, so the corresponding environment variables are not referenced. If more paths are added to the managed directories in the future, they will be derived in the same way from each management directory. The environment variables are colon-separated when multiple directories are specified in a single directive, so this example assumes that each management directory is specified individually.

If you run the script directly without systemd, the systemd environment variables will not be automatically provided. It is usually executed via a unit.

`DOMAINLIST` is assumed to be a single domain in this case.

If you want to specify multiple domains, extend the script to pass multiple `-d` options to Certbot.

## Installation Method

First, create the necessary files and edit the configuration.

```bash
cp certbot.env.example certbot.env
```

Edit the contents of `certbot.env` according to your environment, and then install.

```bash
sudo ./install.sh
```

Check the installed configuration file.

```bash
sudoedit /etc/certbot-renewal/certbot.env
```

Run the initial execution manually.

```bash
sudo systemctl start certbot-renew.service
```

Check the execution log.

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

If you want to check the systemd unit settings, use the following:

```bash
sudo systemd-analyze verify \
    /etc/systemd/system/certbot-renew.service
```

The following can be used to evaluate the permission settings:

```bash
systemd-analyze security certbot-renew.service
```

However, `systemd-analyze security` is a static configuration evaluation and does not guarantee actual attack resistance.

## Migration from Existing Environment

If migrating from an old configuration that used a fixed user, you need to move the existing Certbot persistent data to the new `StateDirectory`.

First, stop the old timer and service, and back up the certificate, private key, ACME account information, and renewal settings.

Simply copying the contents of the old directory may cause the user assigned by `DynamicUser=yes` to not be able to access it.

Migrate, considering systemd's private directory management and ownership processing, and after verifying that it is working, delete the old configuration.

In particular, the symbolic links from `live/` to `archive/` may be broken by the copy method.

## Operational Notes

### Package Updates Every Time

In this configuration, the Certbot Python package update is also checked each time the certificate renewal is attempted.

The introduction of new packages may cause compatibility issues, which may cause certificate renewal to fail.

In continuous operation, it is recommended to monitor systemd's failure logs and be able to quickly detect problems.

Also, the Python package update, which is performed with root privileges, is separate from the ACME communication performed by the non-privileged user.

### Certificate Validity Monitoring

Even if the systemd timer is working correctly, certificate renewal may fail due to DNS, router, UPnP, or Let's Encrypt issues.

It is recommended to monitor the certificate validity in addition to checking that the timer is working.

### Management of Deployment Destination

The certificate directories for Nginx and StrongSwan are managed by root, and write permissions are not granted to non-privileged users.

If more certificate deployment destinations are added, the configuration should be maintained so that the non-privileged Certbot process is not changed, and the root deployment script is added.

### Dependence on External Services

Certificate acquisition depends on Let's Encrypt's ACME service, Python package updates depend on the package index, and UPnP operations depend on the router's implementation.

These failures cannot be completely eliminated, so appropriate logging and failure monitoring is required.

## Summary

In this configuration, systemd's `DynamicUser=yes` is used to execute the certificate renewal process as a non-privileged user, and operations requiring root privileges are separated by attaching the `+` prefix to the `ExecStartPre=` and `ExecStartPost=` commands.

This eliminates the need to create a fixed user and allows systemd to manage ownership and runtime directory management.

Furthermore, Certbot's Python virtual environment is managed by root, and the dependent packages are checked for updates each time using `uv pip` or `pip`.

Installation and removal are managed by `install.sh` and `uninstall.sh`, and the certificate data is retained in a normal uninstall.

The key point of this configuration is to clearly separate the different responsibilities of certificate acquisition, Python package management, and deployment to the production environment, and to grant only the privileges necessary for each process.
