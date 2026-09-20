---
title: "Registering the Legacy Windows Photo Viewer for Picture Files"
description: "Windows Photo Viewer is the classic image viewer that was used in earlier versions of Windows. Some..."
pubDatetime: 2026-06-24T05:59:54.488Z
---

Windows Photo Viewer is the classic image viewer that was used in earlier versions of Windows. Some users still prefer it because it is simple, fast, and familiar. On newer Windows systems, it may not appear as an available app for opening picture files by default.

The registry file shown here is used to register the legacy Windows Photo Viewer as an application for picture file types. After applying it, Windows can show Windows Photo Viewer as an option when choosing an app to open images.

## Purpose

The purpose of this registry entry is to make `photoviewer.dll` available as an image-viewing application in Windows.

It adds Windows Photo Viewer to the current user’s application list and defines how Windows should open and print image files with it. This allows the user to select Windows Photo Viewer from the “Open with” menu or set it as the default app for supported picture file types.

## What It Does

This registry file adds entries under:

`HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll`

Because it uses `HKEY_CURRENT_USER`, the change applies only to the current Windows user account. It does not affect all users on the computer.

The registry entries define two main actions:

## Open Images

The `open` section tells Windows how to launch Windows Photo Viewer when an image file is opened.

It uses `rundll32.exe` to call the Windows Photo Viewer library and open the selected picture in full-screen image-viewing mode.

This makes Windows Photo Viewer available as an app that can open picture files.

## Print Images

The `print` section registers a print action for Windows Photo Viewer.

This allows Windows to recognize that Windows Photo Viewer can also be used when printing image files. The `NeverDefault` value helps prevent this print action from becoming the default print behavior automatically.

## How to Use It

To use this registry content, save it as a `.reg` file.

For example:

`register-photo-viewer.reg`

Then double-click the file and allow Windows to add the information to the registry.

After applying the file, you can choose Windows Photo Viewer for image files by doing the following:

1. Right-click a picture file.
2. Select **Open with**.
3. Choose **Choose another app**.
4. Select **Windows Photo Viewer** if it appears.
5. Optionally enable **Always use this app** to make it the default for that file type.

## Important Note

Editing the Windows Registry can affect how Windows behaves. Before applying a registry file, make sure the contents are trusted and suitable for your system.

This registry file is intended to register the legacy Windows Photo Viewer for the current user only. It does not install a new program; it only makes an existing Windows component available as an application option for picture files.

## Summary

This registry file restores Windows Photo Viewer as a selectable app for image files. It is useful for users who prefer the classic Windows image viewer instead of newer photo apps. Once registered, Windows Photo Viewer can be selected from the “Open with” menu and used to open supported picture files.

```registry
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll]

[HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll\shell]

[HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll\shell\open]
"MuiVerb"="@photoviewer.dll,-3043"

[HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll\shell\open\command]
@=hex(2):25,00,53,00,79,00,73,00,74,00,65,00,6d,00,52,00,6f,00,6f,00,74,00,25,\
  00,5c,00,53,00,79,00,73,00,74,00,65,00,6d,00,33,00,32,00,5c,00,72,00,75,00,\
  6e,00,64,00,6c,00,6c,00,33,00,32,00,2e,00,65,00,78,00,65,00,20,00,22,00,25,\
  00,50,00,72,00,6f,00,67,00,72,00,61,00,6d,00,46,00,69,00,6c,00,65,00,73,00,\
  25,00,5c,00,57,00,69,00,6e,00,64,00,6f,00,77,00,73,00,20,00,50,00,68,00,6f,\
  00,74,00,6f,00,20,00,56,00,69,00,65,00,77,00,65,00,72,00,5c,00,50,00,68,00,\
  6f,00,74,00,6f,00,56,00,69,00,65,00,77,00,65,00,72,00,2e,00,64,00,6c,00,6c,\
  00,22,00,2c,00,20,00,49,00,6d,00,61,00,67,00,65,00,56,00,69,00,65,00,77,00,\
  5f,00,46,00,75,00,6c,00,6c,00,73,00,63,00,72,00,65,00,65,00,6e,00,20,00,25,\
  00,31,00,00,00

[HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll\shell\open\DropTarget]
"Clsid"="{FFE2A43C-56B9-4bf5-9A79-CC6D4285608A}"

[HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll\shell\print]
"NeverDefault"=""

[HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll\shell\print\command]
@=hex(2):25,00,53,00,79,00,73,00,74,00,65,00,6d,00,52,00,6f,00,6f,00,74,00,25,\
  00,5c,00,53,00,79,00,73,00,74,00,65,00,6d,00,33,00,32,00,5c,00,72,00,75,00,\
  6e,00,64,00,6c,00,6c,00,33,00,32,00,2e,00,65,00,78,00,65,00,20,00,22,00,25,\
  00,50,00,72,00,6f,00,67,00,72,00,61,00,6d,00,46,00,69,00,6c,00,65,00,73,00,\
  25,00,5c,00,57,00,69,00,6e,00,64,00,6f,00,77,00,73,00,20,00,50,00,68,00,6f,\
  00,74,00,6f,00,20,00,56,00,69,00,65,00,77,00,65,00,72,00,5c,00,50,00,68,00,\
  6f,00,74,00,6f,00,56,00,69,00,65,00,77,00,65,00,72,00,2e,00,64,00,6c,00,6c,\
  00,22,00,2c,00,20,00,49,00,6d,00,61,00,67,00,65,00,56,00,69,00,65,00,77,00,\
  5f,00,46,00,75,00,6c,00,6c,00,73,00,63,00,72,00,65,00,65,00,6e,00,20,00,25,\
  00,31,00,00,00

[HKEY_CURRENT_USER\SOFTWARE\Classes\Applications\photoviewer.dll\shell\print\DropTarget]
"Clsid"="{60fd46de-f830-4894-a628-6fa81bc0190d}"
```
