---
title: "A Simple Windows Tool to Hide Chrome Picture-in-Picture Behind Other Windows"
description: "Use Python and native Windows APIs to find topmost Chrome windows and remove their pinned stacking behavior without moving, resizing, or activating them."
pubDatetime: 2026-01-20T03:05:36.755Z
---

Chrome’s Picture-in-Picture (PiP) feature is convenient, but on Windows it often behaves like an “always-on-top” window. That means the small floating video stays above other applications even when you would prefer it to sit behind a full-screen app, a document, or a meeting window. The script below is a lightweight Windows utility that removes the “topmost” behavior from Chrome windows—including the PiP window—so you can let it be covered by other windows when needed.

## What the Tool Does

This tool scans all visible windows on your Windows desktop, identifies which ones belong to `chrome.exe`, and then checks whether each Chrome window has the **Topmost** extended window style enabled.

If a Chrome window is marked as topmost, the tool modifies that window so it becomes **not topmost**. In practical terms, this allows Chrome’s PiP window to be hidden behind other windows instead of permanently floating above them.

## How It Works (High-Level)

The script uses Python’s `ctypes` module to call native Windows APIs from `user32.dll` and `kernel32.dll`. The workflow is:

1. **Enumerate visible windows**
   It calls `EnumWindows` to iterate through all top-level windows on the desktop, and filters to those that are visible and have a non-empty window title.

2. **Map each window to a process**
   For each window handle (`HWND`), it calls `GetWindowThreadProcessId` to retrieve the process ID (PID).

3. **Confirm the executable is Chrome**
   With the PID, it opens the process (limited permissions) and uses `QueryFullProcessImageNameW` to retrieve the full path to the executable. It then checks whether the executable name is `chrome.exe`.

4. **Check whether the window is “topmost”**
   It reads the window’s extended style (`GWL_EXSTYLE`) using a 32/64-bit safe approach (`GetWindowLongPtrW` on 64-bit, `GetWindowLongW` on 32-bit), and tests whether the `WS_EX_TOPMOST` flag is present.

5. **Remove the topmost attribute**
   If the flag is present, it calls `SetWindowPos` with `HWND_NOTOPMOST` and a set of safe flags to avoid resizing, moving, or stealing focus.

## Key Windows Concepts Used

### Window Handles (HWND)

Windows identifies each top-level window using a handle. This script collects those handles and uses them to query and modify window properties.

### Extended Window Styles

Windows stores additional behaviors—like “always on top”—in extended style flags. The relevant flag here is:

* `WS_EX_TOPMOST`: indicates a window should remain above most others.

### SetWindowPos and Not-Topmost

The script removes topmost behavior with:

* `SetWindowPos(hwnd, HWND_NOTOPMOST, ...)`

and uses these flags:

* `SWP_NOMOVE` / `SWP_NOSIZE`: do not change the window’s position or size
* `SWP_NOACTIVATE`: do not activate or focus the window
* `SWP_FRAMECHANGED`: forces Windows to refresh certain window attributes

This design minimizes user disruption while changing only the stacking behavior.

## Output and Behavior

When run, the script prints a result line for each Chrome window it successfully updates, for example:

* `[OK] TOPMOST removed ...`

If it fails to modify a window (for permission or other reasons), it prints:

* `[FAIL] Could not modify ... ERROR=...`

At the end, it reports how many windows were modified.

## Practical Notes

* The script targets **all** visible Chrome windows that are topmost, not only the PiP window. This is usually acceptable, but it is worth knowing if you rely on other Chrome windows staying pinned above everything else.
* The tool is conservative about permissions by using `PROCESS_QUERY_LIMITED_INFORMATION`, which helps it work without requiring elevated privileges in many cases.
* Because it filters by executable name (`chrome.exe`), it will not affect other browsers or media players unless you adapt the target executable.

## Summary

This script is a straightforward Windows automation utility: it finds Chrome windows, detects which are set to “always on top,” and removes that behavior. For users who want Chrome Picture-in-Picture to be coverable—rather than permanently floating—this provides a simple, script-based solution using standard Windows APIs via Python.

```python
import ctypes
from ctypes import wintypes
import os
import sys

# Ensure UTF-8 output
try:
    sys.stdout.reconfigure(encoding="utf-8", errors="replace")
except Exception:
    pass

user32 = ctypes.WinDLL("user32", use_last_error=True)
kernel32 = ctypes.WinDLL("kernel32", use_last_error=True)

EnumWindowsProc = ctypes.WINFUNCTYPE(wintypes.BOOL, wintypes.HWND, wintypes.LPARAM)

PROCESS_QUERY_LIMITED_INFORMATION = 0x1000

# --- GWL / style constants ---
GWL_STYLE   = -16
GWL_EXSTYLE = -20

WS_EX_TOPMOST = 0x00000008

# --- SetWindowPos constants ---
HWND_NOTOPMOST = wintypes.HWND(-2)

SWP_NOSIZE       = 0x0001
SWP_NOMOVE       = 0x0002
SWP_NOACTIVATE   = 0x0010
SWP_FRAMECHANGED = 0x0020

# --- function prototypes ---
user32.EnumWindows.argtypes = [EnumWindowsProc, wintypes.LPARAM]
user32.EnumWindows.restype = wintypes.BOOL

user32.IsWindowVisible.argtypes = [wintypes.HWND]
user32.IsWindowVisible.restype = wintypes.BOOL

user32.GetWindowTextLengthW.argtypes = [wintypes.HWND]
user32.GetWindowTextLengthW.restype = ctypes.c_int

user32.GetWindowTextW.argtypes = [wintypes.HWND, wintypes.LPWSTR, ctypes.c_int]
user32.GetWindowTextW.restype = ctypes.c_int

user32.GetWindowThreadProcessId.argtypes = [wintypes.HWND, ctypes.POINTER(wintypes.DWORD)]
user32.GetWindowThreadProcessId.restype = wintypes.DWORD

kernel32.OpenProcess.argtypes = [wintypes.DWORD, wintypes.BOOL, wintypes.DWORD]
kernel32.OpenProcess.restype = wintypes.HANDLE

kernel32.CloseHandle.argtypes = [wintypes.HANDLE]
kernel32.CloseHandle.restype = wintypes.BOOL

kernel32.QueryFullProcessImageNameW.argtypes = [
    wintypes.HANDLE, wintypes.DWORD, wintypes.LPWSTR, ctypes.POINTER(wintypes.DWORD)
]
kernel32.QueryFullProcessImageNameW.restype = wintypes.BOOL

user32.SetWindowPos.argtypes = [
    wintypes.HWND, wintypes.HWND,
    ctypes.c_int, ctypes.c_int, ctypes.c_int, ctypes.c_int,
    wintypes.UINT
]
user32.SetWindowPos.restype = wintypes.BOOL

# --- GetWindowLongPtrW (32/64-bit safe) ---
if ctypes.sizeof(ctypes.c_void_p) == 8:
    user32.GetWindowLongPtrW.argtypes = [wintypes.HWND, ctypes.c_int]
    user32.GetWindowLongPtrW.restype = ctypes.c_longlong
    def get_window_long_ptr(hwnd, index) -> int:
        return int(user32.GetWindowLongPtrW(hwnd, index))
else:
    user32.GetWindowLongW.argtypes = [wintypes.HWND, ctypes.c_int]
    user32.GetWindowLongW.restype = ctypes.c_long
    def get_window_long_ptr(hwnd, index) -> int:
        return int(user32.GetWindowLongW(hwnd, index))


def get_window_title(hwnd: int) -> str:
    length = user32.GetWindowTextLengthW(hwnd)
    if length <= 0:
        return ""
    buffer = ctypes.create_unicode_buffer(length + 1)
    user32.GetWindowTextW(hwnd, buffer, length + 1)
    return buffer.value


def get_process_id(hwnd: int) -> int:
    pid = wintypes.DWORD(0)
    user32.GetWindowThreadProcessId(hwnd, ctypes.byref(pid))
    return int(pid.value)


def get_process_image_path(pid: int) -> str:
    process = kernel32.OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION, False, pid)
    if not process:
        return ""
    try:
        size = wintypes.DWORD(4096)
        buffer = ctypes.create_unicode_buffer(size.value)
        if kernel32.QueryFullProcessImageNameW(process, 0, buffer, ctypes.byref(size)):
            return buffer.value
        return ""
    finally:
        kernel32.CloseHandle(process)


def get_extended_style(hwnd: int) -> int:
    return get_window_long_ptr(hwnd, GWL_EXSTYLE)


def has_flag(value: int, flag: int) -> bool:
    return (value & flag) == flag


def is_target_executable(image_path: str, target_exe: str = "chrome.exe") -> bool:
    if not image_path:
        return False
    return os.path.basename(image_path).lower() == target_exe.lower()


def remove_topmost(hwnd: int) -> None:
    flags = SWP_NOMOVE | SWP_NOSIZE | SWP_NOACTIVATE | SWP_FRAMECHANGED
    if not user32.SetWindowPos(hwnd, HWND_NOTOPMOST, 0, 0, 0, 0, flags):
        raise ctypes.WinError(ctypes.get_last_error())


def enumerate_windows():
    windows = []

    @EnumWindowsProc
    def callback(hwnd, lparam):
        if not user32.IsWindowVisible(hwnd):
            return True

        title = get_window_title(hwnd)
        if not title:
            return True

        pid = get_process_id(hwnd)
        image = get_process_image_path(pid)
        exstyle = get_extended_style(hwnd)

        windows.append((hwnd, pid, title, image, exstyle))
        return True

    if not user32.EnumWindows(callback, 0):
        raise ctypes.WinError(ctypes.get_last_error())

    return windows


if __name__ == "__main__":
    TARGET_EXE = "chrome.exe"
    modified_count = 0

    for hwnd, pid, title, image, exstyle in enumerate_windows():
        if not is_target_executable(image, TARGET_EXE):
            continue

        if has_flag(exstyle, WS_EX_TOPMOST):
            try:
                remove_topmost(hwnd)
                modified_count += 1
                print(f"[OK] TOPMOST removed  HWND=0x{hwnd:08X} PID={pid} TITLE={title} EXE={image}")
            except OSError as e:
                print(f"[FAIL] Could not modify HWND=0x{hwnd:08X} PID={pid} TITLE={title} EXE={image} ERROR={e}")

    print(f"Done: TOPMOST removed from {modified_count} window(s) (target EXE = {TARGET_EXE})")
```
