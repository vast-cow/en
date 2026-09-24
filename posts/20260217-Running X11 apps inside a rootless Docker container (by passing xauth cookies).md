---
title: "Running X11 apps inside a rootless Docker container (by passing xauth cookies)"
description: "Pass an X11 authentication cookie into a rootless container, configure a reachable display, and test GUI access with xev while troubleshooting network and display mismatches."
pubDatetime: 2026-02-17T06:37:00.866Z
---

Rootless Docker is great for reducing host privileges, but GUI apps can be a bit tricky—especially when you want an X11 app inside a container to show up on your host display.

This post walks through a simple, reliable approach:

* **Read the host’s X11 MIT-MAGIC-COOKIE**
* **Register that cookie inside the container with `xauth add`**
* **Launch an X11 client (e.g., `xev`) from the container**

---

## Prerequisites

* An X server is running on the host (Xorg or Xwayland).
* The container can reach the host X server (this guide uses `--network host`).
* You have `xauth` available on the host (and ideally in the container too).

---

## What we’re doing (high-level)

X11 access typically requires:

1. **The correct DISPLAY target** (e.g., `10.0.2.2:10`)
2. **A valid authentication cookie** (MIT-MAGIC-COOKIE-1)

Your host stores these cookies in an Xauthority database. We’ll copy the relevant cookie value from the host and inject it into the container’s Xauthority using `xauth add`.

---

## 1) Get the X11 cookie on the host

On the host, run:

```bash
$ xauth list "$DISPLAY"
hostname/unix:10  MIT-MAGIC-COOKIE-1  0a1b2c...
```

Copy the cookie part (`0a1b2c...`). You’ll use it inside the container.

> Note: the `:10` portion is the display number. Yours might be `:0`, `:1`, etc.

---

## 2) Start the container (rootless Docker)

Example (using an image called `wine`, but any image is fine):

```bash
$ docker run -it --rm --name wine-x11 \
  --network host \
  -e DISPLAY=host.docker.internal:12.0 \
  -v ./root:/root \
  -v ./work:/work \
  wine bash
```

### Why these flags?

* `--network host`
  Easiest way to ensure the container can reach the host X server in many rootless setups.
* `-e DISPLAY=host.docker.internal:12.0`
  Provides a default DISPLAY; we’ll explicitly set the DISPLAY when testing.
* `-v ./root:/root`
  Optional but convenient: persists `/root` so `.Xauthority` can persist too.
* `-v ./work:/work`
  Optional workspace mount.

> If `host.docker.internal` doesn’t resolve in your environment, don’t worry—you can use an explicit host IP (as shown next).

---

## 3) Add the host cookie inside the container with `xauth`

Inside the container shell, register the cookie:

```bash
# xauth add 10.0.2.2:10 MIT-MAGIC-COOKIE-1 "${cookie}"
```

Replace `${cookie}` with the cookie string you copied from the host (`0a1b2c...`).

### About this warning

You may see:

```
xauth:  file /root/.Xauthority does not exist
```

This is **just a warning** meaning the file doesn’t exist yet. `xauth add` will create it, so it’s safe to ignore.

---

## 4) Test with `xev`

Now verify X11 forwarding from container to host:

```bash
# DISPLAY=10.0.2.2:10 xev
```

If a small window opens and prints keyboard/mouse events, it’s working.

---

## Troubleshooting checklist

* **Wrong display number**

  * Ensure `:10` in `10.0.2.2:10` matches what you saw on the host (`xauth list "$DISPLAY"`).
* **Wrong host IP from the container**

  * Confirm `10.0.2.2` is reachable from the container in your setup; it’s environment-dependent.
* **X server not accepting TCP connections**

  * Some environments only allow UNIX socket connections by default.
* **Wayland hosts**

  * You’re likely using Xwayland; DISPLAY numbers often differ from classic `:0`.

---

## Summary

To run an X11 app inside a rootless Docker container:

1. Read the host cookie: `xauth list "$DISPLAY"`
2. Start the container (`--network host` is simplest)
3. Add the cookie in the container: `xauth add ... MIT-MAGIC-COOKIE-1 ...`
4. Launch an X11 client: `DISPLAY=... xev`

This pattern is minimal, explicit, and works well for debugging and quick setups.
