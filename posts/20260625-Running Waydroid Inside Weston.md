---
title: "Running Waydroid Inside Weston"
description: "Launch Android in a nested Weston kiosk session by selecting its Wayland display, then shut down Android and stop the Waydroid session cleanly."
pubDatetime: 2026-06-25T09:59:21.422Z
---

Waydroid can be started inside a Weston session by launching Weston with the Wayland backend, starting the Waydroid session, and then opening the full Waydroid user interface.

This setup is useful when you want to run Waydroid in a controlled Wayland environment instead of directly on the main desktop session.

## Purpose

The purpose of this setup is to run Waydroid inside Weston, a lightweight Wayland compositor. Weston provides a separate graphical environment where Waydroid can display its Android interface.

This can be useful for testing, kiosk-style environments, or systems where you want Waydroid to run in a dedicated display session.

## How to Use It

### 1. Start Weston

First, start Weston using the Wayland backend and kiosk shell:

```bash
weston --backend=wayland-backend.so --shell=kiosk-shell.so --idle-time=0
```

This starts Weston as a nested Wayland compositor. The kiosk shell provides a simple fullscreen-style environment, and `--idle-time=0` prevents Weston from entering an idle state.

### 2. Start the Waydroid Session

Next, start the Waydroid session using the Weston Wayland display:

```bash
WAYLAND_DISPLAY=wayland-1 waydroid session start
```

The `WAYLAND_DISPLAY=wayland-1` setting tells Waydroid to use the Wayland display created by Weston.

### 3. Show the Waydroid Full UI

After the Waydroid session has started, open the full Waydroid interface:

```bash
waydroid show-full-ui
```

This displays the Android interface inside the Weston environment.

## Stopping Waydroid

When you are finished using Waydroid, it is better to shut down the Android environment first and then stop the Waydroid session.

### 1. Shut Down Android Inside Waydroid

Run the following command:

```bash
waydroid shell -- svc power shutdown
```

This sends a shutdown command to the Android system running inside Waydroid. It is similar to powering off an Android device.

### 2. Stop the Waydroid Session

After Android has been shut down, stop the Waydroid session:

```bash
waydroid session stop
```

This stops the active Waydroid session on the host system.

## Notes

The commands should usually be run in order. Weston must be running before starting the Waydroid session, because Waydroid needs the Weston Wayland display to show its interface.

When stopping Waydroid, shut down Android first with `waydroid shell -- svc power shutdown`, then stop the session with `waydroid session stop`.

Depending on your system, the Wayland display name may be different from `wayland-1`. If the command does not work, check the available Wayland display socket and adjust `WAYLAND_DISPLAY` accordingly.

## Summary

Running Waydroid inside Weston is a simple way to launch Android in a dedicated Wayland environment. Start Weston first, point Waydroid to the Weston display, and then open the full Waydroid UI.

To stop it cleanly, shut down Android inside Waydroid first, and then stop the Waydroid session.
