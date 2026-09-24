---
title: "Automatic DDNS Updates with No-IP and systemd"
description: "A Python user service checks the router's public IP through UPnP and updates No-IP when it changes, with persistent state, backups, and journal logging."
pubDatetime: 2026-01-19T14:46:00.254Z
---

Keeping a home or small-office service reachable from the internet can be challenging when your ISP changes your public IP address. A Dynamic DNS (DDNS) service solves this by mapping a hostname to your current public IP, but it still needs a reliable way to detect IP changes and push updates. The setup described here provides a lightweight, automated updater that integrates neatly with `systemd` and only contacts No-IP when an IP change is actually detected.

## Overview of the Approach

This tool is built around a Python script that periodically checks the network’s external (global) IP address and updates a No-IP DDNS hostname only when necessary. It is designed to be resilient—keeping state on disk with backups—and to run continuously in the background as a `systemd` user service, producing logs that can be inspected via `journalctl`.

## Core Functionality

### Detecting the External IP via UPnP

Instead of calling an external “what is my IP” web service, the script retrieves the public IP directly from the router using UPnP, via the `miniupnpc` library. This requires:

* A UPnP-capable router with UPnP enabled.
* Python 3 installed on the machine running the updater.
* The `miniupnpc` Python package available in the environment.

To improve reliability, the script wraps UPnP discovery and IP retrieval in an `IGDClient` helper that:

* Discovers and selects an Internet Gateway Device (IGD).
* Retrieves the external IP through the router.
* Falls back to forced re-discovery if the IGD becomes invalid or returns an empty result.
* Rate-limits repeated discovery attempts to avoid excessive churn during transient failures.

### Updating No-IP Only When the IP Changes

The script maintains the previously known IP address in a local state file (`state.json`). Each loop iteration:

1. Fetches the current external IP via UPnP.
2. Compares it to the last stored IP.
3. If unchanged, does nothing (no No-IP traffic).
4. If changed, sends an authenticated update request to No-IP’s dynamic update endpoint and then persists the new IP to disk.

This “update only on change” behavior reduces unnecessary requests and avoids needless logging noise.

### Safe Local State Persistence with Backup

To avoid losing the last known IP due to partial writes or file corruption, state is written with a simple backup strategy:

* If `state.json` exists, it is renamed to `state_bak.json` before writing a new `state.json`.
* The new state is then written in formatted JSON.

This provides practical crash safety by ensuring a prior copy exists before overwriting state.

## Configuration

### `config.json`

Configuration is kept in `config.json`, which contains No-IP credentials and update metadata:

* `username` / `password`: No-IP account credentials.
* `hostname`: The DDNS hostname to update.
* `useragent`: A descriptive User-Agent string sent with update requests.

A minimal example looks like:

```json
{
  "noip": {
    "username": "username",
    "password": "password",
    "hostname": "all.ddnskey.com",
    "useragent": "user noip-updater Linux user@mail.com"
  }
}
```

The separation between `config.json` (static settings) and `state.json` (mutable runtime state like `myip`) helps keep secrets and operational state organized.

## Running as a systemd User Service

### `noip-updater.service`

The provided `systemd` unit runs the script as a persistent service under the user instance:

* Uses `Type=simple`.
* Writes stdout/stderr into the journal (`StandardOutput=journal`, `StandardError=journal`).
* Identifies logs with a consistent tag (`SyslogIdentifier`).
* Starts the updater via `ExecStart`, and ensures it runs from the correct directory with `WorkingDirectory`.
* Hooks into network readiness with `WantedBy=network-online.target`.

Example:

```ini
[Unit]
Description=noip updater

[Service]
Type=simple
StandardOutput=journal
StandardError=journal
SyslogIdentifier=noip updater
ExecStart=/path/to/python /path/to/noip_updater.py
WorkingDirectory=/path/to/workdir

[Install]
WantedBy=network-online.target
```

This design makes the updater behave like any other managed service: it can be enabled, started at boot, and observed centrally through the journal.

## Installation and Enablement Workflow

The setup uses the `systemd --user` model, with an additional step to make sure user services can persist without an active login session. The documented workflow is:

1. Ensure a bridge exists between system-level boot targets and the user instance (per the referenced guide).
2. Edit configuration and service paths.
3. Install the service file under the user’s systemd directory.
4. Reload, enable, and start the service.
5. Enable linger so the user service can run at boot even when the user is not logged in.

Commands:

```sh
edit config.json
edit noip-updater.service
cp -t ~/.config/systemd/user/ noip-updater.service
systemctl --user daemon-reload
systemctl --user enable --now noip-updater.service
loginctl enable-linger "$USER"
```

## Logging and Operations

Because output is routed to the systemd journal, operational visibility is straightforward. To inspect logs:

```sh
journalctl --user-unit noip-updater.service
```

The script also reduces log spam by suppressing repeated printing of identical exceptions: it normalizes traceback content and only prints when a new error pattern appears. This makes it easier to spot changes in failure modes without wading through redundant entries.

## Summary

This DDNS updater combines three practical ideas:

* A local, router-derived external IP check via UPnP (`miniupnpc`).
* Change-based updates to No-IP using authenticated HTTP requests.
* Reliable, hands-off operation under `systemd` with journal-based logging and state persistence with backups.

The result is a small, maintainable service that keeps a No-IP hostname aligned with your current public IP without unnecessary network calls or manual intervention.


## `noip_updater.py` 
 
```python 
import sys 
import urllib.request 
import urllib.parse 
import json 
import time 
import traceback 
import datetime 
import os 
 
import miniupnpc 
 
CONFIG_FN = "config.json" 
STATE_FN = "state.json" 
 
def _bak_name(path: str) -> str: 
    """ 
    Generate a backup filename by inserting '_bak' before the file extension. 
    """ 
    root, ext = os.path.splitext(path) 
    return f"{root}_bak{ext}" 
 
STATE_BAK_FN = _bak_name(STATE_FN) 
 
def load_json(path: str, default: dict) -> dict: 
    """ 
    Load JSON from disk if it exists; otherwise return a default dictionary. 
    """ 
    if not os.path.exists(path): 
        return default 
    with open(path) as f: 
        return json.load(f) 
 
def atomic_save_json(path: str, bak_path: str, data: dict) -> None: 
    """ 
    Atomically save JSON data by first backing up any existing file. 
    """ 
    # This provides crash safety via backup/rename, but does not guarantee 
    # true atomicity across filesystems or power loss scenarios. 
    if os.path.exists(path): 
        if os.path.exists(bak_path): 
            os.remove(bak_path) 
        os.rename(path, bak_path) 
 
    with open(path, "wt") as f: 
        json.dump(data, f, ensure_ascii=False, indent=2) 
 
def load_config() -> dict: 
    """ 
    Load configuration data (static settings). 
    """ 
    return load_json(CONFIG_FN, default={}) 
 
def load_state() -> dict: 
    """ 
    Load runtime state (mutable data such as last known IP). 
    """ 
    return load_json(STATE_FN, default={"noip": {}}) 
 
def save_state(state: dict) -> None: 
    """ 
    Persist runtime state to disk with backup. 
    """ 
    atomic_save_json(STATE_FN, STATE_BAK_FN, state) 
 
def timestamp_str() -> str: 
    """ 
    Return the current time formatted as HH:MM:SS. 
    """ 
    return datetime.datetime.now().strftime("%H:%M:%S") 
 
NOIP_ENDPOINT = "https://dynupdate.no-ip.com/nic/update" 
 
def noip_build_opener(config: dict): 
    """ 
    Build an HTTP opener configured with HTTP Basic Authentication 
    for the No-IP update endpoint. 
    """ 
    password_mgr = urllib.request.HTTPPasswordMgrWithDefaultRealm() 
 
    password_mgr.add_password( 
        realm=None, 
        uri=NOIP_ENDPOINT, 
        user=config["noip"]["username"], 
        passwd=config["noip"]["password"], 
    ) 
 
    auth_handler = urllib.request.HTTPBasicAuthHandler(password_mgr) 
    opener = urllib.request.build_opener(auth_handler) 
    return opener 
 
def noip_update(config: dict, opener, ipaddr: str): 
    """ 
    Send an update request to No-IP with the current external IP address. 
    """ 
    params = { 
        "hostname": config["noip"]["hostname"], 
        "myip": ipaddr, 
    } 
 
    headers = { 
        "User-Agent": config["noip"]["useragent"], 
    } 
 
    request = urllib.request.Request( 
        NOIP_ENDPOINT + "?" + urllib.parse.urlencode(params), 
        headers=headers, 
    ) 
 
    with opener.open(request) as res: 
        data = res.read() 
 
    return data 
 
class IGDClient: 
    """ 
    Wrapper around miniupnpc to manage IGD discovery and 
    external IP address retrieval with rediscovery control. 
    """ 
 
    def __init__(self, discoverdelay_ms: int = 200, rediscover_cooldown_sec: int = 60): 
        self.discoverdelay_ms = discoverdelay_ms 
        self.rediscover_cooldown_sec = rediscover_cooldown_sec 
        self._u: miniupnpc.UPnP | None = None 
        self._last_discover_ts: float = 0.0 
 
    def _ensure_igd(self, force: bool = False) -> None: 
        """ 
        Ensure that a valid IGD is discovered and selected. 
        Rediscovery is rate-limited unless forced. 
        """ 
        now = time.time() 
 
        # Fast path: reuse existing IGD when allowed 
        if not force and self._u is not None: 
            return 
 
        # Prevent repeated discovery attempts during transient failures 
        if ( 
            not force 
            and (now - self._last_discover_ts) < self.rediscover_cooldown_sec 
            and self._u is not None 
        ): 
            return 
 
        u = miniupnpc.UPnP() 
        u.discoverdelay = self.discoverdelay_ms 
        u.discover() 
        u.selectigd() 
 
        self._u = u 
        self._last_discover_ts = now 
 
    def get_external_ip(self) -> str: 
        """ 
        Retrieve the external IP address via the IGD. 
        Retries with forced rediscovery on failure. 
        """ 
        self._ensure_igd(force=False) 
        try: 
            assert self._u is not None 
            ip = self._u.externalipaddress() 
            if not ip: 
                raise RuntimeError("externalipaddress() returned empty") 
            return ip 
        except Exception: 
            # A forced rediscovery handles cases where the IGD became invalid 
            self._ensure_igd(force=True) 
            assert self._u is not None 
            ip = self._u.externalipaddress() 
            if not ip: 
                raise RuntimeError("externalipaddress() returned empty after rediscover") 
            return ip 
 
if __name__ == "__main__": 
    config = load_config() 
    state = load_state() 
 
    extip_prev = state.get("noip", {}).get("myip") 
    if extip_prev is not None: 
        sys.stderr.write(f"[{timestamp_str()}] myip={extip_prev}\n") 
 
    # Used to suppress repeated logging of the same exception 
    tb_tuple_prev = None 
 
    opener = noip_build_opener(config) 
    igd = IGDClient(discoverdelay_ms=200, rediscover_cooldown_sec=60) 
 
    while True: 
        try: 
            extip = igd.get_external_ip() 
            tb_tuple = None 
 
            if extip != extip_prev: 
                sys.stderr.write(f"[{timestamp_str()}] {extip=}\n") 
 
                response = noip_update(config, opener, extip) 
                sys.stderr.write(f"[{timestamp_str()}] noip {response=}\n") 
 
                state.setdefault("noip", {})["myip"] = extip 
                save_state(state) 
 
        except Exception as e: 
            extip = None 
 
            # Store a normalized traceback representation for comparison 
            tb_tuple = ( 
                "\n".join(traceback.format_tb(e.__traceback__)), 
                repr(e), 
            ) 
 
            if tb_tuple != tb_tuple_prev: 
                sys.stderr.write( 
                    tb_tuple[0] + f"[{timestamp_str()}] " + tb_tuple[1] + "\n" 
                ) 
 
        time.sleep(60) 
        extip_prev = extip 
        tb_tuple_prev = tb_tuple 
```
