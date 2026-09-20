---
title: "Exposing Supermicro X11DPI (ATEN IPMI) with Tailscale Serve: Use `socat` or Nginx `stream` Instead of an HTTP Reverse Proxy"
description: "While trying to expose the IPMI interface (ATEN-based) on a Supermicro X11 series motherboard using..."
pubDatetime: 2026-07-31T11:01:26.884Z
updatedDate: 2026-07-31T11:03:37.278Z
---

While trying to expose the IPMI interface (ATEN-based) on a Supermicro X11 series motherboard using Tailscale Services (`tailscale serve --service`), I ran into an unexpected pitfall.

The conclusion is straightforward:

> **When exposing ATEN IPMI through Tailscale Serve, a TCP-level proxy (such as `socat` or Nginx `stream`) should be your first choice instead of an HTTP reverse proxy.**

## What I Wanted to Do

I wanted to expose the IPMI interface on my LAN under a service name like:

```text
https://x11dpi-ipmi.<tailnet>.ts.net/
```

With Tailscale Services, this can be configured as:

```bash
tailscale serve \
  --service=svc:x11dpi-ipmi \
  --https=443 \
  https+insecure://x.x.x.x
```

However, the HTML5 KVM console did not work correctly.

## My First Suspect: WebSockets

The HTML5 KVM console uses WebSockets.

My initial assumptions were:

* Maybe Tailscale Serve doesn't fully support WebSockets.
* Maybe the `Upgrade` header isn't being forwarded correctly.

However, my own `aiohttp` WebSocket server worked perfectly through the same setup.

That meant there was nothing inherently wrong with the combination of:

* Tailscale
* WebSockets
* The browser

## The Actual Cause

After inserting Nginx as an HTTP reverse proxy for debugging, I found this error:

```text
upstream sent invalid header: "\x20..."
```

resulting in:

```text
502 Bad Gateway
```

The request flow looked like this:

```text
Browser
    ↓
Tailscale
    ↓
Nginx HTTP Proxy
    ↓
ATEN IPMI
```

This indicates that **Nginx rejects the HTTP response headers returned by the ATEN IPMI firmware as invalid.**

Simple GET requests work, but CGI requests such as:

```text
POST /cgi/ipmi.cgi
```

fail.

For example:

```text
op=UID_SUPPORT.XML
```

returns **502 Bad Gateway** when sent via POST.

Meanwhile:

```text
GET /cgi/ipmi.cgi
```

works without issue.

In other words, this is an **HTTP protocol compatibility issue**.

## This Is Not a WebSocket Problem

Since only the HTML5 KVM console initially appeared to fail, WebSockets seemed like the obvious culprit.

In reality, however:

**The HTTP parser fails while processing the CGI POST response.**

The browser never reaches the stage where WebSocket communication becomes relevant because HTTP communication with the IPMI interface has already failed.

## Solution 1: `socat` (Recommended)

Instead of interpreting HTTP at all, simply forward TCP traffic:

```text
Tailscale
    ↓ TLS termination
TCP
    ↓
socat
    ↓ TLS
IPMI
```

For example:

```bash
socat \
  TCP4-LISTEN:8082,bind=127.0.0.1,reuseaddr,fork \
  OPENSSL:x.x.x.x:443,verify=0
```

Then configure Tailscale Serve:

```bash
tailscale serve \
  --service=svc:x11dpi-ipmi \
  --tls-terminated-tcp=443 \
  tcp://127.0.0.1:8082
```

With this configuration, everything worked correctly:

* CGI
* Cookies
* WebSockets
* HTML5 KVM

Because no HTTP headers are parsed, ATEN's non-standard HTTP implementation is passed through unchanged.

## Solution 2: Nginx `stream`

Nginx can achieve the same result by using the `stream` module instead of the HTTP module.

```nginx
stream {
    server {
        listen 127.0.0.1:8082;

        proxy_ssl on;
        proxy_ssl_verify off;

        proxy_pass x.x.x.x:443;
    }
}
```

Since `stream` operates as a TCP proxy, it never parses HTTP headers.

As a result, it avoids compatibility issues with the ATEN IPMI firmware.

## Why I Don't Recommend an HTTP Reverse Proxy

The conventional approach would be something like:

```nginx
proxy_pass https://x.x.x.x;
```

However, with ATEN IPMI this can result in:

```text
upstream sent invalid header
```

This happens **before** considerations such as:

* `Host` headers
* `Origin`
* WebSockets

The HTTP response itself is rejected by Nginx's parser.

Adjusting buffer sizes or disabling buffering (for example, `proxy_buffering off`) does not resolve the problem.

## Why `socat` Works

`socat` does not understand HTTP.

It simply forwards TCP streams.

Therefore, even if the ATEN IPMI firmware returns unconventional HTTP responses, `socat` passes them through unchanged.

Modern browsers are apparently tolerant enough to process those responses successfully.

## Conclusion

When exposing an ATEN-based Supermicro X11 IPMI interface through Tailscale Services, my recommended order is:

1. **`socat` + `tailscale serve --tls-terminated-tcp`**
2. **Nginx `stream` + `tailscale serve --tls-terminated-tcp`**
3. **Nginx HTTP reverse proxy (not recommended)**

For most web applications, an HTTP reverse proxy is the default choice.

ATEN IPMI is an exception.

A TCP-level proxy that forwards traffic without inspecting HTTP is significantly more reliable than an HTTP-aware reverse proxy.

If you encounter errors such as `upstream sent invalid header` or `502 Bad Gateway`, don't assume the problem is with WebSockets or Tailscale. First, check whether you're routing the traffic through an HTTP reverse proxy.
