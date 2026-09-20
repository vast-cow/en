---
title: "Listening for Windows Battery Percentage Changes in C++ with a Hidden Message Window"
description: "A C++ example registers a message-only window for battery notifications, decodes power broadcasts, and releases notification handles when the message loop ends."
pubDatetime: 2026-01-19T08:28:26.802Z
---

This program demonstrates a straightforward way to listen for battery percentage updates on Windows using the Win32 message loop. Instead of polling the battery level repeatedly, it registers for a power setting notification and reacts whenever Windows broadcasts a change.

## What the Program Does

At a high level, the application:

* Creates a **hidden message-only window**.
* Registers for the **battery percentage remaining** power setting (`GUID_BATTERY_PERCENTAGE_REMAINING`).
* Waits in a standard **Win32 message loop**.
* When Windows broadcasts a power setting change, it extracts the new battery percentage and prints it.

The output is a simple console line such as:

* `Battery changed: 73%`

## Build Requirements

The source is intended for MSVC and links against two Windows libraries:

* `user32.lib` for windowing and message handling.
* `powrprof.lib` for power management APIs and GUIDs.

An example compile command is provided:

* `cl /EHsc /W4 /std:c++17 main.cpp user32.lib powrprof.lib`

The file also includes a `#pragma comment(lib, "user32.lib")` directive, and optionally can do the same for `powrprof.lib` if you prefer not to pass it on the command line.

## Key Windows Concepts Used

### Message-Only Window

The program creates a window with `HWND_MESSAGE` as the parent. This produces a **message-only window**, which:

* Is invisible (no UI).
* Exists purely to receive messages.
* Is ideal for background listeners like this.

The class name is defined as a constant:

* `BatteryPercentListenerHiddenWindow`

### WM_POWERBROADCAST and PBT_POWERSETTINGCHANGE

Windows reports many power-related events through `WM_POWERBROADCAST`. This program is specifically interested in `PBT_POWERSETTINGCHANGE`, which indicates that some registered power setting has changed.

When the window procedure receives:

* `msg == WM_POWERBROADCAST`
* `wParam == PBT_POWERSETTINGCHANGE`

…it interprets `lParam` as a `POWERBROADCAST_SETTING*`, which contains:

* `PowerSetting`: the GUID identifying what changed
* `DataLength`: size of the attached data
* `Data`: the new value (raw bytes)

### Battery Percentage Setting GUID

The program filters events to only handle those where:

* `pbs->PowerSetting == GUID_BATTERY_PERCENTAGE_REMAINING`

For this specific GUID, Windows provides a `DWORD` representing the percentage remaining. The code checks `DataLength` before reading:

* `pbs->DataLength >= sizeof(DWORD)`

Then it reads the value and prints it.

## Registering for Notifications

The core subscription happens here:

* `RegisterPowerSettingNotification(hwnd, &GUID_BATTERY_PERCENTAGE_REMAINING, DEVICE_NOTIFY_WINDOW_HANDLE)`

This tells Windows:

* Deliver battery percentage change notifications
* To the specified window handle
* Using the window-handle delivery mechanism (`DEVICE_NOTIFY_WINDOW_HANDLE`)

If registration fails, the program reports the Win32 error via `GetLastError()` and exits cleanly.

## Running the Message Loop

After setup, the program prints a status line and enters the standard Win32 loop:

* `GetMessageA`
* `TranslateMessage`
* `DispatchMessageA`

This keeps the process alive and responsive to system broadcasts. The window procedure handles shutdown via `WM_DESTROY`, calling `PostQuitMessage(0)`.

## Cleanup and Shutdown

On exit (when the message loop ends), the code performs cleanup in the correct order:

* `UnregisterPowerSettingNotification(hNotify)`
* `DestroyWindow(hwnd)`

This ensures the notification handle is released and the message-only window is destroyed.

## Why This Approach Is Useful

This pattern is efficient and idiomatic for Windows desktop code because it:

* Avoids polling and unnecessary CPU wakeups.
* Reacts immediately to system-reported changes.
* Uses a minimal, invisible window rather than a full GUI.

For console utilities, background agents, or monitoring tools that need battery status updates, this is a clean and lightweight solution.

```c++
// Build (MSVC):
//   cl /EHsc /W4 /std:c++17 main.cpp user32.lib

#define WIN32_LEAN_AND_MEAN
#include <windows.h>
#include <powrprof.h>   // GUID_BATTERY_PERCENTAGE_REMAINING
#include <cstdio>

#pragma comment(lib, "user32.lib")

static void OnBatteryPercentChanged(DWORD percent)
{
    std::printf("Battery changed: %lu%%\n", static_cast<unsigned long>(percent));
}

static const char* kWndClassName = "BatteryPercentListenerHiddenWindow";

static LRESULT CALLBACK WndProc(HWND hwnd, UINT msg, WPARAM wParam, LPARAM lParam)
{
    switch (msg)
    {
    case WM_POWERBROADCAST:
        if (wParam == PBT_POWERSETTINGCHANGE)
        {
            auto* pbs = reinterpret_cast<POWERBROADCAST_SETTING*>(lParam);
            if (pbs && pbs->PowerSetting == GUID_BATTERY_PERCENTAGE_REMAINING)
            {
                if (pbs->DataLength >= sizeof(DWORD))
                {
                    DWORD percent = *reinterpret_cast<DWORD*>(pbs->Data);
                    OnBatteryPercentChanged(percent);
                }
            }
        }
        return TRUE;

    case WM_DESTROY:
        PostQuitMessage(0);
        return 0;
    }

    return DefWindowProcA(hwnd, msg, wParam, lParam);
}

int main()
{
    HINSTANCE hInst = GetModuleHandleA(nullptr);

    WNDCLASSEXA wc{};
    wc.cbSize = sizeof(wc);
    wc.lpfnWndProc = WndProc;
    wc.hInstance = hInst;
    wc.lpszClassName = kWndClassName;

    if (!RegisterClassExA(&wc))
    {
        std::fprintf(stderr, "RegisterClassExA failed. err=%lu\n", GetLastError());
        return 1;
    }

    HWND hwnd = CreateWindowExA(
        0,
        kWndClassName,
        "",
        0,
        0, 0, 0, 0,
        HWND_MESSAGE,
        nullptr,
        hInst,
        nullptr);

    if (!hwnd)
    {
        std::fprintf(stderr, "CreateWindowExA failed. err=%lu\n", GetLastError());
        return 1;
    }

    HPOWERNOTIFY hNotify = RegisterPowerSettingNotification(
        hwnd,
        &GUID_BATTERY_PERCENTAGE_REMAINING,
        DEVICE_NOTIFY_WINDOW_HANDLE);

    if (!hNotify)
    {
        std::fprintf(stderr, "RegisterPowerSettingNotification failed. err=%lu\n", GetLastError());
        DestroyWindow(hwnd);
        return 1;
    }

    std::puts("Listening for battery percentage changes. Press Ctrl+C to exit.");

    MSG m{};
    while (GetMessageA(&m, nullptr, 0, 0) > 0)
    {
        TranslateMessage(&m);
        DispatchMessageA(&m);
    }

    UnregisterPowerSettingNotification(hNotify);
    DestroyWindow(hwnd);
    return 0;
}
```
