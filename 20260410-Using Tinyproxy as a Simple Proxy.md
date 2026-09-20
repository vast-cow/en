---
title: "Using Tinyproxy as a Simple Proxy"
description: "This article focuses strictly on setting up and using Tinyproxy—a lightweight HTTP/HTTPS proxy—on..."
pubDatetime: 2026-04-10T07:23:26.713Z
updatedDate: 2026-04-10T07:36:13.895Z
---

This article focuses strictly on setting up and using **Tinyproxy**—a lightweight HTTP/HTTPS proxy—on Alpine Linux. The goal is to get a minimal proxy running and understand how to use it from another machine or client.

---

## 1. Installing Tinyproxy

On Alpine Linux, installation is straightforward:

```sh
apk add tinyproxy
```

This installs the Tinyproxy service and default configuration files.

---

## 2. Basic Configuration

The main configuration file is typically located at:

```conf
/etc/tinyproxy/tinyproxy.conf
```

At minimum, configure the following:

```apache
Port 8080
Listen 0.0.0.0
```

### What these settings mean

* **Port 8080**
  Tinyproxy will listen for incoming proxy connections on port 8080.

* **Listen 0.0.0.0**
  Allows connections from any network interface (not just localhost).
  This is required if you want to access the proxy from another machine.

---

## 3. Starting Tinyproxy

You have two operational modes depending on your use case.

### Option A — Run in foreground (debug mode)

```sh
tinyproxy -d -c /etc/tinyproxy/tinyproxy.conf
```

* `-d`: Runs in foreground (useful for debugging/logging)
* `-c`: Specifies config file

Use this when testing or troubleshooting.

---

### Option B — Run as a background service

```sh
rc-service tinyproxy start
rc-update add tinyproxy
```

* Starts Tinyproxy as a daemon
* Enables it to launch automatically at boot

---

## 4. Using Tinyproxy

Once running, Tinyproxy acts as a standard HTTP proxy.

### Example: Using it from a client

If your Tinyproxy server is at:

```plaintext
172.27.59.40:8080
```

You can configure tools to use it as a proxy.

#### Using `curl`

```sh
curl -x http://172.27.59.40:8080 http://example.com
```

#### Using environment variables

```sh
export http_proxy="http://172.27.59.40:8080"
export https_proxy="http://172.27.59.40:8080"
```

Now most CLI tools will route traffic through Tinyproxy.

---

## 5. Example: Using Tinyproxy with SSH (via netcat)

Tinyproxy can also proxy TCP connections using the HTTP CONNECT method.

Add this to your SSH config:

```ssh
Host foomachine
    HostName 1.2.3.4
    ProxyCommand nc -X connect -x 172.27.59.40:8080 %h %p
```

### Explanation

* `nc -X connect` → uses HTTP CONNECT proxy mode
* `-x 172.27.59.40:8080` → specifies the Tinyproxy server
* `%h %p` → target host and port

Then connect normally:

```sh
ssh foomachine
```

The connection will be routed through Tinyproxy.

---

## Summary

Tinyproxy provides:

* A minimal, lightweight HTTP/HTTPS proxy
* Simple configuration (port + listen address)
* Flexible usage:

  * CLI tools (`curl`, env vars)
  * SSH tunneling via `nc`
  * General proxy routing

For most use cases, the minimal configuration shown above is sufficient to get a working proxy quickly.
