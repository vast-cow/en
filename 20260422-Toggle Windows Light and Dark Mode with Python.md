---
title: "Toggle Windows Light and Dark Mode with Python"
description: "Note: I also created a version that changes the color schemes of VS Code, Windows Terminal and..."
pubDatetime: 2026-04-22T13:05:27.123Z
updatedDate: 2026-06-03T08:39:57.439Z
---

> **Note**: I also created a version that changes the color schemes of VS Code, Windows Terminal and Windows wallpaper (solid color).
> https://dev.to/vast-cow/switch-windows-windows-terminal-and-vs-code-themes-all-at-once-1dae


Switching between light and dark mode on Windows usually requires going through the Settings app. This small Python script simplifies the process by letting you toggle the theme instantly. It works by updating system settings directly and notifying Windows to apply the change right away.

## Purpose

This script is designed to make theme switching quick and automatable. It can be useful in scenarios such as:

* Quickly toggling themes with a single command
* Integrating theme changes into automation scripts
* Setting up time-based switching (e.g., dark mode at night)

## How It Works

The script performs two main actions:

### 1. Modify Windows Registry Settings

Windows stores theme preferences in the user registry. The script updates two values:

* `AppsUseLightTheme` (controls app appearance)
* `SystemUsesLightTheme` (controls system UI appearance)

By setting these values to `1` (light) or `0` (dark), the theme preference is changed.

### 2. Notify the System

After updating the registry, the script sends a system-wide message to inform Windows that the theme has changed. Without this step, the UI would not update immediately.

## Key Functions

### Get the Current Theme

```python
get_current_theme()
```

Returns the current theme setting:

* `1` → Light mode
* `0` → Dark mode

### Set a Specific Theme

```python
set_theme(light: bool)
```

Sets the theme explicitly:

* `True` → Light mode
* `False` → Dark mode

### Toggle the Theme

```python
toggle_theme()
```

Switches the current theme to the opposite state.
If the system is in light mode, it changes to dark mode, and vice versa.

## Usage

### Run the Script

Simply execute the script:

```bash
python script.py
```

Each time it runs, it toggles the current theme and prints the result:

* `Light mode`
* `Dark mode`

### Practical Use Cases

* Assign it to a desktop shortcut for quick access
* Combine it with Task Scheduler for automatic switching
* Integrate it into custom productivity workflows

## Notes

* This script is intended for Windows only
* It modifies the user registry, so appropriate permissions may be required
* Some applications may not reflect the change immediately

## Summary

This approach provides a straightforward way to control Windows appearance settings programmatically. By wrapping registry updates and system notifications into a simple script, it enables fast and flexible theme switching without relying on manual configuration.

```python
import ctypes 
from ctypes import wintypes 
import winreg 
 
 
PERSONALIZE_KEY = "Software\\Microsoft\\Windows\\CurrentVersion\\Themes\\Personalize" 
APPS_KEY = "AppsUseLightTheme" 
SYSTEM_KEY = "SystemUsesLightTheme" 
 
# Win32 constants 
HWND_BROADCAST = 0xFFFF 
WM_SETTINGCHANGE = 0x001A 
SMTO_ABORTIFHUNG = 0x0002 
 
# Fallback for environments where ULONG_PTR is missing 
if hasattr(wintypes, "ULONG_PTR"): 
    ULONG_PTR = wintypes.ULONG_PTR 
else: 
    ULONG_PTR = ctypes.c_size_t 
 
 
def get_current_theme() -> int: 
    try: 
        with winreg.OpenKey(winreg.HKEY_CURRENT_USER, PERSONALIZE_KEY) as key: 
            value, regtype = winreg.QueryValueEx(key, APPS_KEY) 
            if regtype == winreg.REG_DWORD: 
                return int(value) 
    except FileNotFoundError: 
        pass 
    return 1 
 
 
def set_theme(light: bool) -> None: 
    value = 1 if light else 0 
 
    with winreg.CreateKey(winreg.HKEY_CURRENT_USER, PERSONALIZE_KEY) as key: 
        winreg.SetValueEx(key, APPS_KEY, 0, winreg.REG_DWORD, value) 
        winreg.SetValueEx(key, SYSTEM_KEY, 0, winreg.REG_DWORD, value) 
 
    notify_theme_changed() 
 
 
def toggle_theme() -> bool: 
    current = get_current_theme() 
    new_light = not bool(current) 
    set_theme(new_light) 
    return new_light 
 
 
def notify_theme_changed() -> None: 
    user32 = ctypes.WinDLL("user32", use_last_error=True) 
 
    SendMessageTimeoutW = user32.SendMessageTimeoutW 
    SendMessageTimeoutW.argtypes = [ 
        wintypes.HWND, 
        wintypes.UINT, 
        wintypes.WPARAM, 
        wintypes.LPCWSTR, 
        wintypes.UINT, 
        wintypes.UINT, 
        ctypes.POINTER(ULONG_PTR), 
    ] 
    SendMessageTimeoutW.restype = wintypes.LPARAM 
 
    result = ULONG_PTR() 
    ret = SendMessageTimeoutW( 
        HWND_BROADCAST, 
        WM_SETTINGCHANGE, 
        0, 
        "ImmersiveColorSet", 
        SMTO_ABORTIFHUNG, 
        5000, 
        ctypes.byref(result), 
    ) 
 
    if ret == 0: 
        raise ctypes.WinError(ctypes.get_last_error()) 
 
 
if __name__ == "__main__": 
    is_light = toggle_theme() 
    print("Light mode" if is_light else "Dark mode")
```
