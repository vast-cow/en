---
title: "Listening to Battery Percentage Changes on Windows with Python and Win32 Messages"
description: "Monitor battery level and AC/DC changes through Python ctypes callbacks, issue charging reminders, and shut down a hidden Win32 message window cleanly on Ctrl+C."
pubDatetime: 2026-01-19T08:27:11.875Z
updatedDate: 2026-01-19T13:36:10.389Z
---

This script is a Windows-only Python program that monitors battery percentage changes in real time by subscribing to Windows power notifications. It uses `ctypes` to call Win32 APIs directly, creates a hidden message-only window to receive system messages, and prints updates whenever the battery percentage changes. The program is designed to run continuously until the user exits with Ctrl+C (or Ctrl+Break), at which point it performs a clean shutdown.

## Purpose and High-Level Design

Windows reports certain power-related events through the window messaging system. Rather than polling the battery status on an interval, this script uses an event-driven approach:

* Register a hidden window class and create a message-only window.
* Subscribe to the **battery percentage remaining** power setting notification.
* Handle incoming `WM_POWERBROADCAST` messages and extract the new battery percentage.
* Print a message such as `Battery changed: 85%`.
* Exit gracefully when the user presses Ctrl+C.

This design is efficient and responsive because it only runs logic when Windows signals a change.

## Key Windows Messages and Constants

The script defines several Win32 constants that drive its behavior:

* `WM_POWERBROADCAST (0x0218)`: Sent when the system broadcasts power management events.
* `PBT_POWERSETTINGCHANGE (0x8013)`: Indicates a power setting has changed, with detailed data provided.
* `WM_CLOSE (0x0010)`, `WM_DESTROY (0x0002)`: Used to trigger orderly shutdown and end the message loop.

It also uses:

* `HWND_MESSAGE = (HWND)-3`: A special parent that creates a **message-only window**, which is invisible and exists only to receive messages.

## Core Structures and GUID Handling

### GUID for Battery Percentage Remaining

Windows identifies power settings using GUIDs. This script defines the GUID for **battery percentage remaining**:

* `GUID_BATTERY_PERCENTAGE_REMAINING`

A custom `GUID` structure is implemented to match the Win32 GUID layout, and a helper function `make_guid()` populates it. During message handling, GUIDs are compared by converting their raw bytes and checking equality—an effective way to ensure an exact match.

### POWERBROADCAST_SETTING Structure

When `PBT_POWERSETTINGCHANGE` occurs, Windows passes a pointer (`lParam`) to a `POWERBROADCAST_SETTING` structure. The script defines it with:

* `PowerSetting` (GUID)
* `DataLength` (DWORD)
* `Data` (variable-length payload)

Because `Data` is variable-length, the script calculates the correct address of the payload and reads a DWORD from memory to obtain the percentage value.

## Creating a Hidden Message-Only Window

The program registers a custom window class named:

* `BatteryPercentListenerHiddenWindow`

It fills a `WNDCLASSEXW` structure and registers it via `RegisterClassExW`. Then it creates the message-only window with `CreateWindowExW`, using `HWND_MESSAGE` as the parent so the window never appears on screen.

This window exists for one primary reason: to receive `WM_POWERBROADCAST` messages from the operating system.

## Subscribing to Battery Notifications

After creating the hidden window, the script registers for notifications using:

* `RegisterPowerSettingNotification(hwnd, GUID_BATTERY_PERCENTAGE_REMAINING, DEVICE_NOTIFY_WINDOW_HANDLE)`

This tells Windows: “Send power setting change messages to this window whenever the battery percentage remaining changes.”

The function returns a notification handle (`HPOWERNOTIFY`) that must be unregistered during cleanup.

## Handling Power Messages in the Window Procedure

The `WndProc` callback is the heart of the event-driven logic. It checks:

1. Is this a power broadcast message?
2. Is it specifically a power setting change event?
3. Does the power setting GUID match battery percentage remaining?
4. Is the data payload large enough to read a DWORD?

If so, it reads the new percentage and calls:

* `OnBatteryPercentChanged(percent)`

By default, this prints:

* `Battery changed: {percent}%`

This callback is intended as the customization point where you could add business logic, alerts, logging, or integration with other systems.

## Clean Shutdown with Ctrl+C

Windows console Ctrl+C handling does not automatically integrate with a blocked message loop. Since the program spends most of its time inside `GetMessageW` (which blocks), the script explicitly registers a console control handler with:

* `SetConsoleCtrlHandler`

When Ctrl+C or Ctrl+Break is pressed, the handler posts `WM_CLOSE` to the hidden window. The window procedure responds by destroying the window, which triggers `WM_DESTROY`, which posts `WM_QUIT`, which finally ends the message loop.

This creates a clean, deterministic shutdown path that avoids abrupt termination.

## Message Loop and Lifecycle

The script runs a standard Win32 message loop:

* `GetMessageW`
* `TranslateMessage`
* `DispatchMessageW`

It exits when `GetMessageW` returns `0`, indicating `WM_QUIT` has been posted.

## Cleanup and Resource Management

In a `finally` block, the program performs proper teardown:

* Unregister the console control handler.
* Unregister the power setting notification (`UnregisterPowerSettingNotification`).
* Destroy the hidden window (`DestroyWindow`).
* Unregister the window class (`UnregisterClassW`).

This ensures the program does not leave stale registrations or handles behind, even if an exception occurs.

## Summary

This script demonstrates a practical pattern for integrating Python with low-level Windows notifications:

* Use `ctypes` to call Win32 APIs.
* Create a message-only hidden window.
* Subscribe to power setting changes via a GUID.
* Extract and handle battery percentage changes with minimal overhead.
* Exit reliably using a console control handler and standard window teardown.

The result is a small, efficient, event-driven battery monitor suitable for background utilities, telemetry collectors, or any application that needs immediate awareness of battery level changes without polling.


## `main.py`

```python
from power_listener import PowerStatusListener, SYSTEM_POWER_STATUS
import win11toast

def on_battery(prev, cur, sps: SYSTEM_POWER_STATUS | None):
    ac = "?" if sps is None else str(int(sps.ACLineStatus))
    # print(f"[Battery] {prev} -> {cur}%  ACLineStatus={ac}")
    if sps.ACLineStatus == 1 and cur >= 80:
        win11toast.notify(f"battery {cur}%: please stop battery charging", duration='short', button='Dismiss')
    elif sps.ACLineStatus != 1 and cur <= 30:
        win11toast.notify(f"battery {cur}%: please start battery charging", duration='short', button='Dismiss')


def on_power(prev, cur, sps: SYSTEM_POWER_STATUS | None):
    label = {0:"AC(PoAc)",1:"DC(PoDc)",2:"Hot(PoHot)",3:"Short(PoShort)"}.get(cur, f"Unknown({cur})")
    # print(f"[Power] {prev} -> {label}")


def main():
    listener = PowerStatusListener()
    listener.add_battery_percent_changed_handler(on_battery)
    listener.add_power_source_changed_handler(on_power)

    raise SystemExit(listener.run())

if __name__ == '__main__':
    main()
```

## `power_listener.py`

```python
# -*- coding: utf-8 -*-
from __future__ import annotations

import sys
import ctypes
from ctypes import wintypes
from typing import Callable, Optional, List

if sys.platform != "win32":
    raise SystemExit("This module is Windows-only.")

# --- Win32 DLLs ---
user32 = ctypes.WinDLL("user32", use_last_error=True)
kernel32 = ctypes.WinDLL("kernel32", use_last_error=True)

# --- Constants ---
WM_POWERBROADCAST = 0x0218
PBT_POWERSETTINGCHANGE = 0x8013
WM_DESTROY = 0x0002
WM_CLOSE = 0x0010

DEVICE_NOTIFY_WINDOW_HANDLE = 0x00000000
HWND_MESSAGE = wintypes.HWND(-3)

CTRL_C_EVENT = 0
CTRL_BREAK_EVENT = 1

GWLP_USERDATA = -21  # user data index for Get/SetWindowLongPtr

kWndClassName = "PowerStatusListenerHiddenWindow"

# --- Types / Structs ---
LRESULT = wintypes.LPARAM
WNDPROC = ctypes.WINFUNCTYPE(LRESULT, wintypes.HWND, wintypes.UINT, wintypes.WPARAM, wintypes.LPARAM)


class GUID(ctypes.Structure):
    _fields_ = [
        ("Data1", wintypes.DWORD),
        ("Data2", wintypes.WORD),
        ("Data3", wintypes.WORD),
        ("Data4", wintypes.BYTE * 8),
    ]


def make_guid(data1, data2, data3, d4_bytes):
    g = GUID()
    g.Data1 = data1
    g.Data2 = data2
    g.Data3 = data3
    g.Data4[:] = (wintypes.BYTE * 8)(*d4_bytes)
    return g


# Battery percentage remaining
GUID_BATTERY_PERCENTAGE_REMAINING = make_guid(
    0xA7AD8041, 0xB45A, 0x4CAE,
    (0x87, 0xA3, 0xEE, 0xCB, 0xB4, 0x68, 0xA9, 0xE1)
)

# AC/DC power source (PoAc/PoDc)
GUID_ACDC_POWER_SOURCE = make_guid(
    0x5D3E9A59, 0xE9D5, 0x4B00,
    (0xA6, 0xBD, 0xFF, 0x34, 0xFF, 0x51, 0x65, 0x48)
)


class POWERBROADCAST_SETTING(ctypes.Structure):
    _fields_ = [
        ("PowerSetting", GUID),
        ("DataLength", wintypes.DWORD),
        ("Data", wintypes.BYTE * 1),
    ]


HCURSOR = getattr(wintypes, "HCURSOR", wintypes.HANDLE)
HICON = getattr(wintypes, "HICON", wintypes.HANDLE)
HBRUSH = getattr(wintypes, "HBRUSH", wintypes.HANDLE)
HMENU = getattr(wintypes, "HMENU", wintypes.HANDLE)


class WNDCLASSEXW(ctypes.Structure):
    _fields_ = [
        ("cbSize", wintypes.UINT),
        ("style", wintypes.UINT),
        ("lpfnWndProc", WNDPROC),
        ("cbClsExtra", ctypes.c_int),
        ("cbWndExtra", ctypes.c_int),
        ("hInstance", wintypes.HINSTANCE),
        ("hIcon", HICON),
        ("hCursor", HCURSOR),
        ("hbrBackground", HBRUSH),
        ("lpszMenuName", wintypes.LPCWSTR),
        ("lpszClassName", wintypes.LPCWSTR),
        ("hIconSm", HICON),
    ]


class POINT(ctypes.Structure):
    _fields_ = [("x", wintypes.LONG), ("y", wintypes.LONG)]


class MSG(ctypes.Structure):
    _fields_ = [
        ("hwnd", wintypes.HWND),
        ("message", wintypes.UINT),
        ("wParam", wintypes.WPARAM),
        ("lParam", wintypes.LPARAM),
        ("time", wintypes.DWORD),
        ("pt", POINT),
    ]


HPOWERNOTIFY = wintypes.HANDLE

# --- SYSTEM_POWER_STATUS ---
class SYSTEM_POWER_STATUS(ctypes.Structure):
    _fields_ = [
        ("ACLineStatus", ctypes.c_byte),
        ("BatteryFlag", ctypes.c_byte),
        ("BatteryLifePercent", ctypes.c_byte),
        ("Reserved1", ctypes.c_byte),
        ("BatteryLifeTime", ctypes.c_ulong),
        ("BatteryFullLifeTime", ctypes.c_ulong),
    ]


kernel32.GetSystemPowerStatus.restype = ctypes.c_bool
kernel32.GetSystemPowerStatus.argtypes = [ctypes.POINTER(SYSTEM_POWER_STATUS)]


def get_system_power_status() -> Optional[SYSTEM_POWER_STATUS]:
    sps = SYSTEM_POWER_STATUS()
    ok = kernel32.GetSystemPowerStatus(ctypes.byref(sps))
    return sps if ok else None


# --- Function prototypes ---
kernel32.GetModuleHandleW.argtypes = [wintypes.LPCWSTR]
kernel32.GetModuleHandleW.restype = wintypes.HMODULE

user32.RegisterClassExW.argtypes = [ctypes.POINTER(WNDCLASSEXW)]
user32.RegisterClassExW.restype = wintypes.ATOM

user32.UnregisterClassW.argtypes = [wintypes.LPCWSTR, wintypes.HINSTANCE]
user32.UnregisterClassW.restype = wintypes.BOOL

user32.CreateWindowExW.argtypes = [
    wintypes.DWORD, wintypes.LPCWSTR, wintypes.LPCWSTR, wintypes.DWORD,
    ctypes.c_int, ctypes.c_int, ctypes.c_int, ctypes.c_int,
    wintypes.HWND, wintypes.HMENU, wintypes.HINSTANCE, wintypes.LPVOID
]
user32.CreateWindowExW.restype = wintypes.HWND

user32.DestroyWindow.argtypes = [wintypes.HWND]
user32.DestroyWindow.restype = wintypes.BOOL

user32.DefWindowProcW.argtypes = [wintypes.HWND, wintypes.UINT, wintypes.WPARAM, wintypes.LPARAM]
user32.DefWindowProcW.restype = LRESULT

user32.PostQuitMessage.argtypes = [ctypes.c_int]
user32.PostQuitMessage.restype = None

user32.PostMessageW.argtypes = [wintypes.HWND, wintypes.UINT, wintypes.WPARAM, wintypes.LPARAM]
user32.PostMessageW.restype = wintypes.BOOL

user32.GetMessageW.argtypes = [ctypes.POINTER(MSG), wintypes.HWND, wintypes.UINT, wintypes.UINT]
user32.GetMessageW.restype = ctypes.c_int

user32.TranslateMessage.argtypes = [ctypes.POINTER(MSG)]
user32.TranslateMessage.restype = wintypes.BOOL

user32.DispatchMessageW.argtypes = [ctypes.POINTER(MSG)]
user32.DispatchMessageW.restype = LRESULT

user32.RegisterPowerSettingNotification.argtypes = [wintypes.HANDLE, ctypes.POINTER(GUID), wintypes.DWORD]
user32.RegisterPowerSettingNotification.restype = HPOWERNOTIFY

user32.UnregisterPowerSettingNotification.argtypes = [HPOWERNOTIFY]
user32.UnregisterPowerSettingNotification.restype = wintypes.BOOL

# WindowLongPtr
user32.SetWindowLongPtrW.argtypes = [wintypes.HWND, ctypes.c_int, wintypes.LPARAM]
user32.SetWindowLongPtrW.restype = wintypes.LPARAM
user32.GetWindowLongPtrW.argtypes = [wintypes.HWND, ctypes.c_int]
user32.GetWindowLongPtrW.restype = wintypes.LPARAM

# Ctrl handler
PHANDLER_ROUTINE = ctypes.WINFUNCTYPE(wintypes.BOOL, wintypes.DWORD)
kernel32.SetConsoleCtrlHandler.argtypes = [PHANDLER_ROUTINE, wintypes.BOOL]
kernel32.SetConsoleCtrlHandler.restype = wintypes.BOOL

# --- Callback type aliases ---
BatteryPercentCallback = Callable[[Optional[int], int, Optional[SYSTEM_POWER_STATUS]], None]
PowerSourceCallback = Callable[[Optional[int], int, Optional[SYSTEM_POWER_STATUS]], None]


class PowerStatusListener:
    """
    Uses a hidden Windows message-only window to monitor:
      - Battery remaining percentage (%)
      - AC/DC power source
    and notifies user-registered callbacks.

    run(): starts the (blocking) message loop
    stop(): requests shutdown by posting WM_CLOSE
    """

    def __init__(self) -> None:
        self._self_pyobj = ctypes.py_object(self)
        self.hInstance = kernel32.GetModuleHandleW(None)
        self.hwnd = wintypes.HWND(0)

        self._wndproc = WNDPROC(self._WndProcThunk)
        self._ctrl_handler = PHANDLER_ROUTINE(self._ConsoleCtrlHandlerThunk)

        self._hNotifies: List[HPOWERNOTIFY] = []

        self._prev_percent: Optional[int] = None
        self._prev_power_source: Optional[int] = None  # 0=AC, 1=DC, 2=Hot, 3=Short (environment-dependent)

        self._battery_cbs: List[BatteryPercentCallback] = []
        self._power_cbs: List[PowerSourceCallback] = []

    # ---- public API (registration) ----
    def add_battery_percent_changed_handler(self, cb: BatteryPercentCallback) -> None:
        self._battery_cbs.append(cb)

    def remove_battery_percent_changed_handler(self, cb: BatteryPercentCallback) -> None:
        self._battery_cbs = [x for x in self._battery_cbs if x is not cb]

    def add_power_source_changed_handler(self, cb: PowerSourceCallback) -> None:
        self._power_cbs.append(cb)

    def remove_power_source_changed_handler(self, cb: PowerSourceCallback) -> None:
        self._power_cbs = [x for x in self._power_cbs if x is not cb]

    # Aliases for compatibility if you prefer "OnXxx" naming
    OnBatteryPercentChanged = add_battery_percent_changed_handler
    OnPowerSourceChanged = add_power_source_changed_handler

    # ---- lifecycle ----
    def run(self) -> int:
        self._register_class()
        try:
            self._create_window()
            self._install_ctrl_handler()
            self._register_power_notify()
            self._message_loop()
            return 0
        finally:
            self._cleanup()

    def stop(self) -> None:
        if self.hwnd:
            user32.PostMessageW(self.hwnd, WM_CLOSE, 0, 0)

    # ---- internal helpers ----
    def _register_class(self) -> None:
        wc = WNDCLASSEXW()
        wc.cbSize = ctypes.sizeof(WNDCLASSEXW)
        wc.style = 0
        wc.lpfnWndProc = self._wndproc
        wc.cbClsExtra = 0
        wc.cbWndExtra = 0
        wc.hInstance = self.hInstance
        wc.hIcon = None
        wc.hCursor = None
        wc.hbrBackground = None
        wc.lpszMenuName = None
        wc.lpszClassName = kWndClassName
        wc.hIconSm = None

        atom = user32.RegisterClassExW(ctypes.byref(wc))
        if not atom:
            raise ctypes.WinError(ctypes.get_last_error())

    def _create_window(self) -> None:
        hwnd = user32.CreateWindowExW(
            0, kWndClassName, "",
            0,
            0, 0, 0, 0,
            HWND_MESSAGE, None, self.hInstance, None
        )
        if not hwnd:
            raise ctypes.WinError(ctypes.get_last_error())

        self.hwnd = hwnd

        # Associate `self` with hwnd
        self_ptr = ctypes.addressof(self._self_pyobj)
        user32.SetWindowLongPtrW(self.hwnd, GWLP_USERDATA, wintypes.LPARAM(self_ptr))

    def _install_ctrl_handler(self) -> None:
        if not kernel32.SetConsoleCtrlHandler(self._ctrl_handler, True):
            raise ctypes.WinError(ctypes.get_last_error())

    def _register_power_notify(self) -> None:
        for guid in (GUID_BATTERY_PERCENTAGE_REMAINING, GUID_ACDC_POWER_SOURCE):
            h = user32.RegisterPowerSettingNotification(
                self.hwnd,
                ctypes.byref(guid),
                DEVICE_NOTIFY_WINDOW_HANDLE
            )
            if not h:
                raise ctypes.WinError(ctypes.get_last_error())
            self._hNotifies.append(h)

    def _message_loop(self) -> None:
        msg = MSG()
        while True:
            r = user32.GetMessageW(ctypes.byref(msg), None, 0, 0)
            if r == 0:
                break
            if r == -1:
                raise ctypes.WinError(ctypes.get_last_error())
            user32.TranslateMessage(ctypes.byref(msg))
            user32.DispatchMessageW(ctypes.byref(msg))

    def _cleanup(self) -> None:
        # Disable Ctrl handler
        if getattr(self, "_ctrl_handler", None):
            kernel32.SetConsoleCtrlHandler(self._ctrl_handler, False)

        # Unregister power notifications
        for h in getattr(self, "_hNotifies", []):
            if h:
                user32.UnregisterPowerSettingNotification(h)
        if hasattr(self, "_hNotifies"):
            self._hNotifies.clear()

        # Destroy window
        if self.hwnd:
            user32.DestroyWindow(self.hwnd)
            self.hwnd = wintypes.HWND(0)

        # Unregister window class
        if self.hInstance:
            user32.UnregisterClassW(kWndClassName, self.hInstance)

    # ---- dispatch helpers ----
    def _emit_battery_percent(self, prev: Optional[int], cur: int, sps: Optional[SYSTEM_POWER_STATUS]) -> None:
        for cb in list(self._battery_cbs):
            try:
                cb(prev, cur, sps)
            except Exception:
                # Prevent callback exceptions from crashing the listener.
                # Replace with logging if needed.
                pass

    def _emit_power_source(self, prev: Optional[int], cur: int, sps: Optional[SYSTEM_POWER_STATUS]) -> None:
        for cb in list(self._power_cbs):
            try:
                cb(prev, cur, sps)
            except Exception:
                pass

    # ---- thunks (ctypes callback) ----
    def _ConsoleCtrlHandlerThunk(self, ctrl_type: int) -> bool:
        if ctrl_type in (CTRL_C_EVENT, CTRL_BREAK_EVENT):
            self.stop()
            return True
        return False

    def _WndProcThunk(self, hwnd, msg, wParam, lParam):
        ptr = user32.GetWindowLongPtrW(hwnd, GWLP_USERDATA)
        if ptr:
            obj: "PowerStatusListener" = ctypes.cast(ptr, ctypes.POINTER(ctypes.py_object)).contents.value
        else:
            obj = self  # fallback for early messages

        if msg == WM_POWERBROADCAST and wParam == PBT_POWERSETTINGCHANGE:
            pbs = ctypes.cast(lParam, ctypes.POINTER(POWERBROADCAST_SETTING)).contents

            if pbs.DataLength >= ctypes.sizeof(wintypes.DWORD):
                data_addr = ctypes.addressof(pbs) + POWERBROADCAST_SETTING.Data.offset
                value = int(wintypes.DWORD.from_address(data_addr).value)
                sps = get_system_power_status()

                if bytes(pbs.PowerSetting) == bytes(GUID_BATTERY_PERCENTAGE_REMAINING):
                    prev = obj._prev_percent
                    obj._prev_percent = value
                    obj._emit_battery_percent(prev, value, sps)
                    return 1

                if bytes(pbs.PowerSetting) == bytes(GUID_ACDC_POWER_SOURCE):
                    prev = obj._prev_power_source
                    obj._prev_power_source = value
                    obj._emit_power_source(prev, value, sps)
                    return 1

            return 1

        if msg == WM_CLOSE:
            user32.DestroyWindow(hwnd)
            return 0

        if msg == WM_DESTROY:
            user32.PostQuitMessage(0)
            return 0

        return user32.DefWindowProcW(hwnd, msg, wParam, lParam)
```
