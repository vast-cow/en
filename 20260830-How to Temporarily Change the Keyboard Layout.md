---
title: "How to Temporarily Change the Keyboard Layout"
description: "In a Linux desktop environment, you can temporarily change the keyboard layout using the setxkbmap..."
pubDatetime: 2026-08-30T07:29:13.210Z
---

In a Linux desktop environment, you can temporarily change the keyboard layout using the `setxkbmap` command.

This is useful if you want to use a US keyboard layout or switch back to a Japanese layout.

## Changing to the US Layout

To change the keyboard layout to the US layout, run the following command:

```bash
setxkbmap us
```

This will use the US layout in the current login session.

## Changing to the Japanese Layout

To change to the Japanese layout, run the following command:

```bash
setxkbmap jp
```

This can also be used to switch back to the Japanese layout after changing to the US layout.

## Checking the Current Settings

To check the keyboard layout currently in use, run the following command:

```bash
setxkbmap -query
```

You can check the current layout by looking at the `layout` item in the displayed output.

For example, it will display `us` for the US layout or `jp` for the Japanese layout.

## Note that the Change is Temporary

Changes made with `setxkbmap` are generally temporary.

Logging out or restarting may revert to the original keyboard layout set by the distribution or desktop environment.

If you want to use the same keyboard layout after logging in, you need to configure a permanent setting. The configuration method varies depending on the distribution you are using, such as Ubuntu, Xubuntu, Debian, or Arch Linux.

If you just want to temporarily switch layouts, it is convenient to remember the three commands: `setxkbmap us`, `setxkbmap jp`, and `setxkbmap -query`.
