---
title: "How to Use Windows Mobile Hotspot Without an Internet Connection"
description: "Purpose   This program is designed to make Windows always pass its internet connectivity..."
pubDatetime: 2026-07-03T08:34:16.519Z
---

## Purpose

This program is designed to make Windows always pass its internet connectivity check.

When Windows connects to a network, it performs a check to determine whether the network can actually reach the internet. If this check fails, Windows considers the network to have "No Internet."

As a result, some features, such as Mobile Hotspot, may not work properly in environments without an internet connection.

By using this program, you can make Windows successfully complete its connectivity check on the local PC. As a result, even in an environment without actual internet access, Windows will consider the system to be connected to the internet, making it possible to use Mobile Hotspot more easily.

## How It Works

This program mainly performs the following two tasks.

### 1. Change the Windows Connectivity Check Host to localhost

Windows normally uses `www.msftconnecttest.com` for its internet connectivity check.

This program temporarily modifies the Windows registry so that the connectivity check is performed against `localhost` instead.

`localhost` refers to the local computer itself. This allows the connectivity check to be answered locally without requiring access to the external internet.

### 2. Start a Web Server for the Connectivity Check

The program starts a simple web server on the local PC.

When Windows accesses `/connecttest.txt`, the program returns the following content.

```text
Microsoft Connect Test
```

This is the response that Windows expects. As a result, Windows determines that the internet connectivity check has succeeded.

## Basic Usage

## 1. Run as Administrator

Because this program modifies the Windows registry, it must be run with administrator privileges.

Launch Command Prompt or PowerShell as an administrator and run the Python script.

## 2. Make Sure Port 80 Is Available

The program starts its web server on `port 80`.

If another web server or application is already using port 80, the program may fail to start. In that case, stop the application using port 80 before running the program.

## 3. Enable Mobile Hotspot

With the program running, enable Windows Mobile Hotspot.

Since Windows considers the internet connectivity check successful, Mobile Hotspot may be available even on networks without internet access.

## Behavior on Exit

When the program exits, it attempts to restore the modified registry value.

At startup, it changes the connectivity check host to:

```text
localhost
```

When the program exits, it restores the original host:

```text
www.msftconnecttest.com
```

The restoration process is performed whenever possible, not only during a normal exit but also when the program is terminated with Ctrl+C or a similar signal.

## Notes

This program makes Windows believe that its internet connectivity check has succeeded.

It does **not** provide actual internet access to external websites or online services. Its sole purpose is to satisfy Windows' connectivity check.

Because it modifies the registry, it should be used with care. If the program terminates unexpectedly, the registry setting may not be restored automatically. In that case, you must manually change the `ActiveWebProbeHost` value back to `www.msftconnecttest.com`.

## Summary

This program allows Windows to successfully complete its internet connectivity test by responding locally on the PC.

As a result, Windows can recognize the system as having an active internet connection even when no actual internet connection is available.

Its primary use case is enabling Windows Mobile Hotspot in environments without internet access. Although the mechanism is relatively simple, it is useful for features that depend on Windows' internet connectivity detection.

```python
#!/usr/bin/env python3
import atexit
import signal
import sys
import winreg
from aiohttp import web

CONNECT_TEST_BODY = b"Microsoft Connect Test"
NOT_FOUND_BODY = b"Page not found"

REDIRECT_LOCATION = "http://go.microsoft.com/fwlink/?LinkID=219472&clcid=0x409"

REG_PATH = r"SYSTEM\CurrentControlSet\Services\NlaSvc\Parameters\Internet"
REG_VALUE_NAME = "ActiveWebProbeHost"

STARTUP_PROBE_HOST = "localhost"
SHUTDOWN_PROBE_HOST = "www.msftconnecttest.com"


def set_active_web_probe_host(value: str) -> None:
    with winreg.OpenKey(
        winreg.HKEY_LOCAL_MACHINE,
        REG_PATH,
        0,
        winreg.KEY_SET_VALUE,
    ) as key:
        winreg.SetValueEx(
            key,
            REG_VALUE_NAME,
            0,
            winreg.REG_SZ,
            value,
        )


def restore_active_web_probe_host() -> None:
    try:
        set_active_web_probe_host(SHUTDOWN_PROBE_HOST)
        print(f"Restored {REG_VALUE_NAME} = {SHUTDOWN_PROBE_HOST}")
    except Exception as e:
        print(f"Failed to restore registry value: {e}", file=sys.stderr)


def setup_registry() -> None:
    set_active_web_probe_host(STARTUP_PROBE_HOST)
    print(f"Set {REG_VALUE_NAME} = {STARTUP_PROBE_HOST}")

    atexit.register(restore_active_web_probe_host)


def handle_exit_signal(signum, frame) -> None:
    restore_active_web_probe_host()
    sys.exit(0)


async def connecttest(request: web.Request) -> web.Response:
    return web.Response(
        status=200,
        body=CONNECT_TEST_BODY,
        headers={
            "Content-Type": "text/plain",
            "Cache-Control": "max-age=30, must-revalidate",
            "Connection": "keep-alive",
        },
    )


async def redirect(request: web.Request) -> web.Response:
    return web.Response(
        status=302,
        reason="Moved Temporarily",
        body=b"",
        headers={
            "Server": "AkamaiGHost",
            "Content-Length": "0",
            "Location": REDIRECT_LOCATION,
            "Connection": "keep-alive",
        },
    )


async def not_found(request: web.Request) -> web.Response:
    return web.Response(
        status=404,
        body=NOT_FOUND_BODY,
        headers={
            "Content-Type": "text/plain",
            "Connection": "keep-alive",
        },
    )


def create_app() -> web.Application:
    app = web.Application()

    app.router.add_get("/connecttest.txt", connecttest)
    app.router.add_get("/redirect", redirect)

    app.router.add_route("*", "/{tail:.*}", not_found)

    return app


if __name__ == "__main__":
    setup_registry()

    signal.signal(signal.SIGINT, handle_exit_signal)
    signal.signal(signal.SIGTERM, handle_exit_signal)

    web.run_app(
        create_app(),
        host="0.0.0.0",
        port=80,
        access_log=None,
    )
```
