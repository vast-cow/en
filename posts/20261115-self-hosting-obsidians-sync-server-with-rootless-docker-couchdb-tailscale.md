---
title: "Self-hosting Obsidian's sync server with Rootless Docker + CouchDB + Tailscale"
description: "It uses Rootless Docker to run CouchDB and Tailscale Serve for HTTPS."
pubDatetime: 2026-11-15T20:50:00+09:00
---

## Obsidian Self-Hosting with Rootless Docker, CouchDB, and Tailscale

### Description

A self-hosted LiveSync configuration using CouchDB as the synchronization server.

### Publication Date

2026-11-15T20:50:00+09:00

---

This guide outlines how to set up an environment for synchronizing notes between your iPhone and PC without using Obsidian's official synchronization service, "Obsidian Sync."

We will use CouchDB as the synchronization server and connect it to Obsidian using the Self-hosted LiveSync plugin.

To ensure security and ease of management, we will adopt the following configuration:

- Run CouchDB with **Rootless Docker**.
- Start CouchDB using **`docker run`** instead of Docker Compose.
- Use the **Host Network** for Docker's network.
- Mount CouchDB's data and configuration files from a directory on the host machine.
- Pre-calculate the UID/GID mapping for Rootless Docker and set file ownership accordingly.
- Run Tailscale on the host OS and use **Tailscale Serve to terminate HTTPS**.
- CouchDB will listen on `127.0.0.1:5984`.
- Each device, including the iPhone, will synchronize via Tailnet.

## 1. System Architecture

The communication flow is as follows:

```text
iPhone / PC / Mac
    │
    │ Obsidian
    │ Self-hosted LiveSync
    │
    ▼
Tailscale Network
    │
    │ HTTPS :443
    ▼
Linux Host
    │
    ├── Tailscale Serve
    │       │
    │       │ HTTP
    │       ▼
    │   127.0.0.1:5984
    │       │
    │       ▼
    └── Rootless Docker
            │
            │ --network host
            ▼
        CouchDB
            │
            ├── local.ini
            └── data/
```

CouchDB will not be directly exposed to the internet.

By using Tailscale Serve, only devices participating in the Tailnet can access CouchDB via HTTPS.

## 2. Prerequisites

The following environment is assumed:

| Item           | Condition                                   |
|----------------|---------------------------------------------|
| OS             | Linux (Ubuntu, Debian, etc.)                |
| Docker         | Rootless Docker                             |
| Docker Engine  | 29.5 or later                              |
| Startup Method | `docker run`                                |
| CouchDB        | Official image `couchdb:3`                   |
| Network        | `--network host`                            |
| Data Storage   | Bind mount to a host directory               |
| VPN            | Tailscale                                   |
| HTTPS          | Tailscale Serve                             |
| Client         | Obsidian + Self-hosted LiveSync              |

**Reason for requiring Docker Engine 29.5 or later:**

Older versions of Rootless Docker, when using `--network host`, may use the RootlessKit's internal network namespace, preventing access to the container's localhost from the actual host OS.

This limitation has been resolved in Docker Engine 29.5 or later.

Check the Docker version:

```bash
docker version --format '{{.Server.Version}}'
```

Confirm that Rootless Docker is running:

```bash
docker info --format '{{json .SecurityOptions}}'
```

Verify that the output contains `rootless`.

If you have multiple Docker contexts, select Rootless Docker:

```bash
docker context use rootless
```

The Docker context name may vary depending on your environment.

## 3. Create a Working Directory

Create a directory for CouchDB in your home directory:

```bash
mkdir -p ~/obsidian-livesync/data
cd ~/obsidian-livesync
```

The final directory structure will be:

```text
~/obsidian-livesync/
├── .env
├── local.ini
└── data/
```

The purpose of each file is as follows:

| File     | Purpose                                 |
|----------|-----------------------------------------|
| `.env`   | CouchDB administrator credentials       |
| `local.ini` | CouchDB configuration                   |
| `data/`  | Persistent storage for database files |

Docker Named Volumes will not be used.

## 4. Understand Rootless Docker's UID/GID Mapping

The official CouchDB Docker image uses user ID (UID) 5984 within the container.

However, in Rootless Docker, the UID within the container does not match the UID on the host OS.

For example, if your `/etc/subuid` looks like this:

```text
user:100000:65536
```

In a standard Rootless Docker setup, the mapping is as follows:

| Container UID | Host UID |
|---|---|
| 0 | Docker execution user's UID |
| 1 | 100000 |
| 2 | 100001 |
| 1000 | 100999 |
| 5984 | 105983 |

The formula is:

```text
Host UID = subuid start value + Container UID - 1
```

In this example:

```text
100000 + 5984 - 1 = 105983
```

The GID is calculated similarly using `/etc/subgid`.

However, the subuid and subgid start values are not always the same.

Also, this formula assumes the standard configuration where container ID 0 is assigned to the host's execution user, and IDs 1 and above are assigned from the subordinate ID range.

### Automatically Calculate UID/GID

Use the following Bash script to calculate the host's UID and GID:

```bash
#!/usr/bin/env bash
set -euo pipefail

CONTAINER_UID=5984
CONTAINER_GID=5984

USER_NAME=$(id -un)

read -r SUBUID_START SUBUID_COUNT < <(
  awk -F: -v user="$USER_NAME" \
    '$1 == user {print $2, $3; exit}' /etc/subuid
)

read -r SUBGID_START SUBGID_COUNT < <(
  awk -F: -v user="$USER_NAME" \
    '$1 == user {print $2, $3; exit}' /etc/subgid
)

if [[ -z "${SUBUID_START:-}" ||
      -z "${SUBGID_START:-}" ]]; then
  echo "Subordinate UID/GID not found" >&2
  exit 1
fi

if (( CONTAINER_UID > SUBUID_COUNT ||
      CONTAINER_GID > SUBGID_COUNT )); then
  echo "UID/GID is outside the mapped range" >&2
  exit 1
fi

HOST_UID=$((SUBUID_START + CONTAINER_UID - 1))
HOST_GID=$((SUBGID_START + CONTAINER_GID - 1))

printf 'Host UID: %s\n' "$HOST_UID"
printf 'Host GID: %s\n' "$HOST_GID"
```

For example, if both the subuid and subgid start values are 100000, the output will be:

```text
Host UID: 105983
Host GID: 105983
```

These values vary depending on the environment, so do not treat them as fixed values.

## 5. Create the CouchDB Configuration File

Create `local.ini`:

```bash
nano ~/obsidian-livesync/local.ini
```

The contents should be:

```ini
[couchdb]
single_node = true
max_document_size = 50000000

[cluster]
n = 1

[chttpd]
bind_address = 127.0.0.1
port = 5984
require_valid_user = true
enable_cors = true
max_http_request_size = 4294967296

[chttpd_auth]
require_valid_user = true

[httpd]
WWW-Authenticate = Basic realm="couchdb"

[cors]
credentials = true
origins = app://obsidian.md,capacitor://localhost,http://localhost
headers = accept,authorization,content-type,origin,referer
methods = GET,PUT,POST,HEAD,DELETE
max_age = 3600

[log]
level = warning
```

Here's an explanation of the key configuration items:

### single_node

```ini
[couchdb]
single_node = true
```

Run CouchDB in single-node mode.

This setting automatically creates the necessary system databases on startup.

### bind_address

```ini
[chttpd]
bind_address = 127.0.0.1
```

Make CouchDB listen only on the host's loopback address.

This is important because we are using Host Network.

Using `0.0.0.0` would cause CouchDB to listen on the host's external network interface as well.

### require_valid_user

```ini
[chttpd]
require_valid_user = true

[chttpd_auth]
require_valid_user = true
```

Reject unauthenticated access.

### CORS

```ini
[cors]
credentials = true
origins = app://obsidian.md,capacitor://localhost,http://localhost
```

This is the configuration required by Self-hosted LiveSync to access from Obsidian's desktop and mobile apps.

### 6. Create CouchDB Administrator Credentials

Create `.env`:

```bash
cd ~/obsidian-livesync

umask 077

printf 'COUCHDB_USER=admin\nCOUCHDB_PASSWORD=%s\n' \
  "$(openssl rand -hex 24)" > .env
```

The contents will be in the following format:

```dotenv
COUCHDB_USER=admin
COUCHDB_PASSWORD=randomly generated password
```

Set file permissions:

```bash
chmod 600 .env
```

This password will be used later in Obsidian's Self-hosted LiveSync.

Do not commit `.env` to Git or other version control systems.

## 7. Set File Ownership for Bind Mounts

Adjust the ownership of `data/` and `local.ini` to match Rootless Docker's UID/GID mapping.

The following is an example for an environment using standard single subuid/subgid range:

```bash
cd ~/obsidian-livesync

CONTAINER_UID=5984
CONTAINER_GID=5984

USER_NAME=$(id -un)

read -r SUBUID_START SUBUID_COUNT < <(
  awk -F: -v user="$USER_NAME" \
    '$1 == user {print $2, $3; exit}' /etc/subuid
)

read -r SUBGID_START SUBGID_COUNT < <(
  awk -F: -v user="$USER_NAME" \
    '$1 == user {print $2, $3; exit}' /etc/subgid
)

if [[ -z "${SUBUID_START:-}" ||
      -z "${SUBGID_START:-}" ]]; then
  echo "Subordinate UID/GID not found" >&2
  exit 1
fi

if (( CONTAINER_UID > SUBUID_COUNT ||
      CONTAINER_GID > SUBGID_COUNT )); then
  echo "UID/GID is outside the mapped range" >&2
  exit 1
fi

HOST_UID=$((SUBUID_START + CONTAINER_UID - 1))
HOST_GID=$((SUBGID_START + CONTAINER_GID - 1))

sudo chown -R "$HOST_UID:$HOST_GID" ./data
sudo chown "$HOST_UID:$HOST_GID" ./local.ini

sudo chmod 700 ./data
sudo chmod 644 ./local.ini
```

This ensures that the container's UID/GID 5984:5984 has the corresponding ownership on the host.

If you have existing data, back it up before changing ownership.

**Note:** This method assumes the standard Rootless Docker user namespace. If you are using custom ID mappings or multiple subordinate ID ranges, you need to check the actual `uid_map` and `gid_map`.

## 8. Get the CouchDB Docker Image

Get the official CouchDB Docker image:

```bash
docker pull couchdb:3
```

We will use CouchDB version 3 in this example.

In a production environment, it is recommended to fix the version tag or digest after testing to prevent unintended version updates.

## 9. Start CouchDB with `docker run`

Execute the following command:

```bash
cd ~/obsidian-livesync

docker run -d \
  --name obsidian-couchdb \
  --restart unless-stopped \
  --network host \
  --env-file "$PWD/.env" \
  --env ERL_EPMD_ADDRESS=127.0.0.1 \
  --env "ERL_FLAGS=-kernel inet_dist_use_interface '{127,0,0,1}'" \
  --mount type=bind,src="$PWD/data",dst=/opt/couchdb/data \
  --mount type=bind,src="$PWD/local.ini",dst=/opt/couchdb/etc/local.ini,readonly \
  couchdb:3
```

The options have the following roles:

| Option       | Description                                        |
|--------------|----------------------------------------------------|
| `-d`         | Run in the background                               |
| `--name`     | Container name                                      |
| `--restart unless-stopped` | Automatically restart on Docker startup            |
| `--network host` | Use the host network                              |
| `--env-file` | Load administrator credentials                     |
| `ERL_EPMD_ADDRESS` | Restrict Erlang's port mapper to loopback           |
| `ERL_FLAGS` | Restrict Erlang node-to-node communication interface |
| `--mount`    | Bind mount data and configuration files              |

The data directory will be writable, and the configuration file will be mounted as read-only.

Do not use `sudo` when operating with Rootless Docker.

### Note: Erlang-related ports

CouchDB uses ports related to Erlang's node management and communication in addition to HTTP port 5984.

Since we are using Host Network, we also need to confirm that these ports are not exposed externally.

In addition to the environment variables specified, check the standby address using the `ss` command below.

## 10. Verify CouchDB Startup

Check if the container is running:

```bash
docker ps --filter name=obsidian-couchdb
```

Check the logs:

```bash
docker logs --tail 100 obsidian-couchdb
```

Run a health check on CouchDB:

```bash
curl http://127.0.0.1:5984/_up
```

If it is running correctly, you should get the following response:

```json
{"status":"ok"}
```

Also, confirm the administrator authentication:

```bash
curl -u admin \
  http://127.0.0.1:5984/
```

If authentication is successful, CouchDB information will be returned.

## 11. Create the Obsidian Database

Create a database for synchronization:

```bash
curl --fail-with-body \
  -u admin \
  -X PUT \
  http://127.0.0.1:5984/obsidiannotes
```

If successful, you will receive the following response:

```json
{"ok":true}
```

If a database already exists, an error indicating that will be returned.

Check the list of databases:

```bash
curl -u admin \
  http://127.0.0.1:5984/_all_dbs
```

If the single-node setup is working correctly, all databases, including the system databases, will be returned.

## 12. Verify Network Standby Status

Since we are using Host Network, confirm that CouchDB is not listening on external interfaces.

```bash
sudo ss -lntp | grep -E ':(5984|4369|9100)\b'
```

For port 5984, the following standby status is desirable:

```text
127.0.0.1:5984
```

If it is `0.0.0.0:5984`, check the configuration.

Also, confirm that Erlang-related ports such as 4369 and 9100 are not exposed externally.

## 13. Set up Tailscale Serve on the Host OS

Tailscale will be run on the host OS, not inside the Docker container.

First, check if Tailscale is connected to the Tailnet:

```bash
sudo tailscale status
```

Next, set up Tailscale Serve:

```bash
sudo tailscale serve --bg \
  http://127.0.0.1:5984
```

This will cause Tailscale to accept HTTPS connections and forward them to CouchDB on the host.

Check the setup status:

```bash
sudo tailscale serve status
```

An HTTPS URL like the following will be assigned:

```text
https://livesync.example.ts.net
```

This is an example URL. Use the URL assigned to your Tailnet.

If you have not configured HTTPS certificates for Tailscale Serve, enable MagicDNS and HTTPS in the Tailscale control panel.

**Do not use Tailscale Funnel.**

Serve is used to limit access to devices within the Tailnet and prevent general public exposure.

## 14. Verify HTTPS Connectivity

Check from another device connected to Tailscale:

```bash
curl https://livesync.example.ts.net/_up
```

If successful, the following response will be received:

```json
{"status":"ok"}
```

Also, check endpoints that require authentication:

```bash
curl -u admin \
  https://livesync.example.ts.net/
```

Finally, check the CORS headers:

```bash
curl -i \
  -H "Origin: app://obsidian.md" \
  -u admin \
  https://livesync.example.ts.net/
```

Confirm that the following header is included in the response:

```text
Access-Control-Allow-Origin: app://obsidian.md
```

If there is a problem with CORS, review the `local.ini` settings.

## 15. Connect Obsidian on PC

Install Obsidian on your PC or Mac.

Then, install and enable the **Self-hosted LiveSync** community plugin.

In the settings, select CouchDB as the synchronization destination.

| Setting Item     | Value                                    |
|------------------|------------------------------------------|
| Remote Type      | CouchDB                                  |
| URI              | `https://livesync.example.ts.net`          |
| Username         | `admin`                                  |
| Password         | Password generated in `.env`              |
| Database name    | `obsidiannotes`                          |
| End-to-End Encryption | Recommended to enable                  |

The E2EE passphrase should be set separately from the CouchDB administrator password.

If you are synchronizing an existing vault, back it up first.

When synchronizing to a new, empty database or joining an existing database, the initialization operation is different, so check the plugin setup screen and select the appropriate option.

## 16. Connect Obsidian on iPhone

Install the following apps on your iPhone:

- Obsidian
- Tailscale

Log in to Tailscale and connect to the same Tailnet as your PC.

In Obsidian, create a new, empty vault and enable Self-hosted LiveSync.

Generate a setup URI for the plugin on your PC and transfer it to your iPhone.

The setup URI contains connection information, so transfer it securely and set a protection passphrase.

On the iPhone, select the option to join an existing synchronization environment and retrieve notes from the server.

After setting up, create and edit test notes on both your PC and iPhone to confirm that they are synchronized in both directions.

Due to iOS's background execution limitations, constant background synchronization is not guaranteed. To ensure synchronization, launch Obsidian and check the synchronization status.

## 17. Container Management

Daily operations are performed using the `docker` command.

### Start

```bash
docker start obsidian-couchdb
```

### Stop

```bash
docker stop obsidian-couchdb
```

### Restart

```bash
docker restart obsidian-couchdb
```

### Check Logs

```bash
docker logs -f obsidian-couchdb
```

### Remove Container

```bash
docker rm -f obsidian-couchdb
```

Removing the container will not remove the `~/obsidian-livesync/data/` directory, which is bind-mounted.

You can resume the data by recreating the container with the same directory mounted.

## 18. Automatic Startup of Rootless Docker

If you want Rootless Docker to start after the host restarts, enable the user service in systemd.

```bash
sudo loginctl enable-linger "$USER"
```

Then, execute the following:

```bash
systemctl --user enable --now docker
```

Since `--restart unless-stopped` is specified when running `docker run`, the container should automatically restart after the Docker daemon starts.

## 19. Backup

CouchDB's data is stored in a directory on the host:

```text
~/obsidian-livesync/data/
```

When performing a normal file backup, stop CouchDB first to ensure the integrity of the database files.

```bash
docker stop obsidian-couchdb
```

Then, perform the backup:

```bash
sudo tar -czf \
  "$HOME/obsidian-couchdb-backup.tar.gz" \
  -C "$HOME/obsidian-livesync" \
  data
```

After completion, restart it:

```bash
docker start obsidian-couchdb
```

Also, back up the administrator credentials and configuration file separately.

Note that synchronization is not a substitute for backups. Back up the Obsidian vault itself on your PC as well.

## 20. Troubleshooting

### Permission denied error

Check the Rootless Docker's UID/GID mapping:

```bash
cat /etc/subuid
cat /etc/subgid
```

The actual mapping within the container can be checked as follows:

```bash
PID=$(docker inspect -f '{{.State.Pid}}' obsidian-couchdb)

sudo cat "/proc/$PID/uid_map"
sudo cat "/proc/$PID/gid_map"
```

If the mapping is different than expected, reset the ownership of `data/` and `local.ini`.

### Unable to connect to CouchDB

Check from the host:

```bash
curl http://127.0.0.1:5984/_up
```

If this fails, first check the CouchDB startup status and `local.ini`.

Also, check the Rootless Docker version and Host Network operation.

### Can connect on PC but not on iPhone

Check the following:

- Is Tailscale connected on the iPhone?
- Is the Tailscale Serve HTTPS URL correct?
- Are the CouchDB CORS settings correct?
- Do the Obsidian connection details match?

### Authentication error during synchronization

Compare the administrator credentials in `.env` with the credentials registered in Obsidian's Self-hosted LiveSync.

Also, verify that CouchDB configuration changes have been applied correctly.

## 21. Final Configuration Summary

| Item                    | Configuration                                   |
|-------------------------|-------------------------------------------------|
| Application             | Obsidian                                        |
| Synchronization Plugin | Self-hosted LiveSync                            |
| Synchronization Server  | Apache CouchDB                                  |
| Container Runtime       | Rootless Docker                                 |
| Container Startup       | `docker run`                                    |
| Network                 | Host Network                                    |
| CouchDB Port            | `127.0.0.1:5984`                               |
| HTTPS Termination       | Host OS's Tailscale Serve                       |
| VPN                     | Tailscale                                       |
| Data Storage            | Bind mount to a host directory                   |
| Configuration File      | `local.ini`                                    |
| UID/GID                 | Calculated from `/etc/subuid` and `/etc/subgid` |
| Docker Compose          | Not used                                        |
| Docker Named Volume     | Not used                                        |
| iOS Support             | Yes                                             |
| Software Costs          | Free                                            |

With this configuration, you can manage your Obsidian synchronization data on your own server and access it via HTTPS from your iPhone, PC, and Mac.

By running CouchDB with Rootless Docker and limiting the network exposure to Tailscale, you avoid the need for a publicly accessible reverse proxy or a VPN container within Docker.

However, there are environment-specific differences in Rootless Docker's user namespace, Host Network behavior, and CouchDB permission settings. Always verify the standby address, ownership, and synchronization operation after the initial setup.

---

## References

- [Docker Official: Rootless mode](https://docs.docker.com/engine/security/rootless/)
- [Docker Official: Limitations and Troubleshooting of Rootless mode](https://docs.docker.com/engine/security/rootless/troubleshoot/)
- [Apache CouchDB Official: Docker Installation](https://docs.couchdb.org/en/stable/install/docker.html)
- [Apache CouchDB Official: Single Node Setup](https://docs.couchdb.org/en/stable/setup/single-node.html)
- [Self-hosted LiveSync Official Repository](https://github.com/vrtmrz/obsidian-livesync)
- [Tailscale Official: Serve](https://tailscale.com/docs/features/tailscale-serve)
