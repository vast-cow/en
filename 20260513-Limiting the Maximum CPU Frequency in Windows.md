---
title: "Limiting the Maximum CPU Frequency in Windows"
description: "I looked into how to limit the maximum CPU frequency on a Panasonic Let’s note CF-FV5USVCP running..."
pubDatetime: 2026-05-13T06:54:42.085Z
updatedDate: 2026-05-13T07:34:17.798Z
---

I looked into how to limit the maximum CPU frequency on a Panasonic Let’s note CF-FV5USVCP running Windows. The conclusion was that it was not enough to configure only the standard power plan. I also had to configure Windows’ **overlay power schemes** and the settings for **multiple CPU efficiency classes**, which appear on Core Ultra-generation CPUs.

The settings that finally worked were as follows:

```powershell
$ACMHz = 1500
$DCMHz = 1500

$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)

foreach ($scheme in $Schemes) {
  foreach ($setting in $FreqSettings) {
    powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz 2>$null
    powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz 2>$null
  }
}

powercfg /setactive SCHEME_CURRENT
````

In this example, the maximum CPU frequency is limited to **1500 MHz** both when plugged in and when running on battery.

---

## Background

The CF-FV5USVCP uses an Intel Core Ultra-generation CPU, which has multiple types of cores, such as P-cores, E-cores, and Low Power E-cores. With this kind of CPU, simply setting “Maximum processor state” to a certain percentage may not reduce the frequency as expected.

Windows’ `powercfg` command provides settings that let you specify the maximum CPU frequency in MHz.

The main settings are:

```text
PROCFREQMAX
PROCFREQMAX1
PROCFREQMAX2
```

On my system, I confirmed this with the following command:

```powershell
powercfg /qh SCHEME_CURRENT SUB_PROCESSOR | findstr /i "PROCFREQMAX PROCTHROTTLEMAX PERFEPP PERFBOOSTMODE"
```

The output included the following:

```text
PERFEPP
PERFEPP1
PERFEPP2
PROCFREQMAX
PROCFREQMAX1
PROCFREQMAX2
PROCTHROTTLEMAX
PROCTHROTTLEMAX1
PROCTHROTTLEMAX2
PERFBOOSTMODE
```

This means that, on this system, not only `PROCFREQMAX` but also `PROCFREQMAX1` and `PROCFREQMAX2` should be configured.

---

## The Initial Problem

At first, I configured only the current power plan like this:

```powershell
powercfg /setacvalueindex SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX 3000
powercfg /setdcvalueindex SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX 2200
powercfg /setactive SCHEME_CURRENT
```

However, the setting did not seem to take effect as expected.

When I checked the active power plan, it turned out to be a Panasonic-specific one:

```powershell
powercfg /getactivescheme
```

Output:

```text
Power Scheme GUID: a83ffe77-647e-45df-899e-cd3f4e2835a1  (Panasonic Power Management)
```

Windows also has power mode overlays in addition to the normal power plan.

When I checked them, `OVERLAY_SCHEME_MIN` had values configured, but `OVERLAY_SCHEME_MAX` did not.

```powershell
powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX
```

Output:

```text
Current AC Power Setting Index: 0x00000000
Current DC Power Setting Index: 0x00000000
```

`0x00000000` can effectively be treated as “no limit.”

On the other hand, `OVERLAY_SCHEME_MIN` did have values:

```powershell
powercfg /q OVERLAY_SCHEME_MIN SUB_PROCESSOR PROCFREQMAX
```

Output:

```text
Current AC Power Setting Index: 0x00001194
Current DC Power Setting Index: 0x00000bb8
```

Converted from hexadecimal to decimal, these mean:

```text
0x00001194 = 4500 MHz
0x00000bb8 = 3000 MHz
```

In other words, the “Better battery life” overlay had frequency limits configured, while the “Best performance” overlay remained unlimited.

If Windows is set to a performance-oriented power mode, `OVERLAY_SCHEME_MAX` may be used. This means that changing only `SCHEME_CURRENT`, or only `OVERLAY_SCHEME_MIN`, may not be enough.

---

## The Settings That Finally Worked

In the end, I configured all of the following.

Target schemes:

```text
SCHEME_CURRENT
OVERLAY_SCHEME_MIN
OVERLAY_SCHEME_MAX
OVERLAY_SCHEME_HIGH
```

Target settings:

```text
PROCFREQMAX
PROCFREQMAX1
PROCFREQMAX2
```

The script I used was:

```powershell
$ACMHz = 1500
$DCMHz = 1500

$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)

foreach ($scheme in $Schemes) {
  foreach ($setting in $FreqSettings) {
    powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz 2>$null
    powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz 2>$null
  }
}

powercfg /setactive SCHEME_CURRENT
```

After this, the 1500 MHz limit appeared to apply both when plugged in and when running on battery.

---

## What Each Command Does

### `$ACMHz` / `$DCMHz`

```powershell
$ACMHz = 1500
$DCMHz = 1500
```

These specify the maximum CPU frequency in MHz for AC power and battery power.

For example:

```powershell
$ACMHz = 3000
$DCMHz = 2200
```

If you want to strongly reduce heat and fan noise, 1500–2200 MHz is a reasonable range to try. If you want to preserve more performance, 2500–3500 MHz may be a better starting point.

### `$Schemes`

```powershell
$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)
```

This lists the power schemes to configure.

| Scheme                | Meaning                         |
| --------------------- | ------------------------------- |
| `SCHEME_CURRENT`      | The currently active power plan |
| `OVERLAY_SCHEME_MIN`  | The power-saving overlay        |
| `OVERLAY_SCHEME_MAX`  | The maximum-performance overlay |
| `OVERLAY_SCHEME_HIGH` | A high-performance overlay      |

In this case, configuring only `SCHEME_CURRENT` was not enough. Since `OVERLAY_SCHEME_MAX` was still unlimited, the frequency limit likely did not apply when Windows was using a performance-oriented power mode.

### `$FreqSettings`

```powershell
$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)
```

This lists the CPU frequency limit settings to configure.

On Core Ultra-generation CPUs, multiple settings may appear for different efficiency classes. Since `PROCFREQMAX2` existed on this system, I configured all three settings.

### `setacvalueindex` / `setdcvalueindex`

```powershell
powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz
powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz
```

These commands mean:

| Command             | Meaning                             |
| ------------------- | ----------------------------------- |
| `/setacvalueindex`  | Sets the value used when plugged in |
| `/setdcvalueindex`  | Sets the value used on battery      |
| `SUB_PROCESSOR`     | Processor power management          |
| `$setting`          | A setting such as `PROCFREQMAX`     |
| `$ACMHz` / `$DCMHz` | The maximum frequency in MHz        |

### `2>$null`

```powershell
2>$null
```

This discards error output.

Some settings, such as `PROCFREQMAX2` or `OVERLAY_SCHEME_HIGH`, may not exist or may not be accessible on every system. In those cases, errors may be shown, so this suppresses them.

However, during initial testing, it is better not to use `2>$null`, because it can hide configuration failures.

For testing, run a command like this first:

```powershell
powercfg /setacvalueindex OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX 1500
```

Once you have confirmed that it works, you can add `2>$null` when running the full script.

### `powercfg /setactive`

```powershell
powercfg /setactive SCHEME_CURRENT
```

This reactivates the current power plan after changing the settings.

It is run at the end to help apply the updated configuration.

---

## How to Check the Settings

After applying the settings, you can check them with:

```powershell
powercfg /q SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX
powercfg /q SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX1
powercfg /q SCHEME_CURRENT SUB_PROCESSOR PROCFREQMAX2

powercfg /q OVERLAY_SCHEME_MIN SUB_PROCESSOR PROCFREQMAX
powercfg /q OVERLAY_SCHEME_MIN SUB_PROCESSOR PROCFREQMAX1
powercfg /q OVERLAY_SCHEME_MIN SUB_PROCESSOR PROCFREQMAX2

powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX
powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX1
powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX2
```

If you set the limit to 1500 MHz, the value should appear as:

```text
0x000005dc
```

The hexadecimal value `0x000005dc` is 1500 in decimal.

For 2200 MHz:

```text
0x00000898
```

For 3000 MHz:

```text
0x00000bb8
```

For 4500 MHz:

```text
0x00001194
```

---

## How to Restore the Default Behavior

To remove the limit, set the value to `0`.

```powershell
$ACMHz = 0
$DCMHz = 0

$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)

foreach ($scheme in $Schemes) {
  foreach ($setting in $FreqSettings) {
    powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz 2>$null
    powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz 2>$null
  }
}

powercfg /setactive SCHEME_CURRENT
```

For `PROCFREQMAX` settings, `0` effectively means “unlimited.”

---

## Python Script to Reapply the Settings After Resume

The `powercfg` settings above are generally retained, but depending on the environment, Windows power mode or manufacturer-specific power management software may appear to revert the behavior after resuming from sleep or hibernation.

For that reason, I also prepared a Python script that detects resume events and automatically reapplies the CPU frequency limit.

This script applies the `powercfg` settings once at startup, then waits for Windows suspend/resume notifications. When the system resumes from sleep or hibernation, it reapplies the same `PROCFREQMAX` settings.

```python
import ctypes
import ctypes.wintypes
import subprocess
import threading
import time
import atexit


if ctypes.sizeof(ctypes.c_void_p) == 8:
    ctypes.wintypes.LRESULT = ctypes.c_longlong
else:
    ctypes.wintypes.LRESULT = ctypes.c_long


# =========================
# Settings
# =========================

ACMHz = 1500
DCMHz = 1500

SCHEMES = [
    "SCHEME_CURRENT",
    "OVERLAY_SCHEME_MIN",
    "OVERLAY_SCHEME_MAX",
    "OVERLAY_SCHEME_HIGH",
]

FREQ_SETTINGS = [
    "PROCFREQMAX",
    "PROCFREQMAX1",
    "PROCFREQMAX2",
]


# =========================
# Windows constants
# =========================

CREATE_NO_WINDOW = 0x08000000

DEVICE_NOTIFY_CALLBACK = 0x00000002

PBT_APMSUSPEND = 0x0004
PBT_APMRESUMESUSPEND = 0x0007
PBT_APMRESUMEAUTOMATIC = 0x0012

WM_QUIT = 0x0012

CTRL_C_EVENT = 0
CTRL_BREAK_EVENT = 1
CTRL_CLOSE_EVENT = 2
CTRL_LOGOFF_EVENT = 5
CTRL_SHUTDOWN_EVENT = 6


# =========================
# ctypes definitions
# =========================

DEVICE_NOTIFY_CALLBACK_ROUTINE = ctypes.WINFUNCTYPE(
    ctypes.wintypes.ULONG,
    ctypes.c_void_p,
    ctypes.wintypes.ULONG,
    ctypes.c_void_p,
)


class DEVICE_NOTIFY_SUBSCRIBE_PARAMETERS(ctypes.Structure):
    _fields_ = [
        ("Callback", DEVICE_NOTIFY_CALLBACK_ROUTINE),
        ("Context", ctypes.c_void_p),
    ]


class MSG(ctypes.Structure):
    _fields_ = [
        ("hwnd", ctypes.wintypes.HWND),
        ("message", ctypes.wintypes.UINT),
        ("wParam", ctypes.wintypes.WPARAM),
        ("lParam", ctypes.wintypes.LPARAM),
        ("time", ctypes.wintypes.DWORD),
        ("pt", ctypes.wintypes.POINT),
    ]


powrprof = ctypes.WinDLL("powrprof", use_last_error=True)
user32 = ctypes.WinDLL("user32", use_last_error=True)
kernel32 = ctypes.WinDLL("kernel32", use_last_error=True)


powrprof.PowerRegisterSuspendResumeNotification.restype = ctypes.wintypes.DWORD
powrprof.PowerRegisterSuspendResumeNotification.argtypes = [
    ctypes.wintypes.DWORD,
    ctypes.POINTER(DEVICE_NOTIFY_SUBSCRIBE_PARAMETERS),
    ctypes.POINTER(ctypes.wintypes.HANDLE),
]

powrprof.PowerUnregisterSuspendResumeNotification.restype = ctypes.wintypes.DWORD
powrprof.PowerUnregisterSuspendResumeNotification.argtypes = [
    ctypes.wintypes.HANDLE,
]

user32.GetMessageW.restype = ctypes.wintypes.BOOL
user32.GetMessageW.argtypes = [
    ctypes.POINTER(MSG),
    ctypes.wintypes.HWND,
    ctypes.wintypes.UINT,
    ctypes.wintypes.UINT,
]

user32.TranslateMessage.restype = ctypes.wintypes.BOOL
user32.TranslateMessage.argtypes = [
    ctypes.POINTER(MSG),
]

user32.DispatchMessageW.restype = ctypes.wintypes.LRESULT
user32.DispatchMessageW.argtypes = [
    ctypes.POINTER(MSG),
]

user32.PostThreadMessageW.restype = ctypes.wintypes.BOOL
user32.PostThreadMessageW.argtypes = [
    ctypes.wintypes.DWORD,
    ctypes.wintypes.UINT,
    ctypes.wintypes.WPARAM,
    ctypes.wintypes.LPARAM,
]

kernel32.GetCurrentThreadId.restype = ctypes.wintypes.DWORD
kernel32.GetCurrentThreadId.argtypes = []

PHANDLER_ROUTINE = ctypes.WINFUNCTYPE(
    ctypes.wintypes.BOOL,
    ctypes.wintypes.DWORD,
)

kernel32.SetConsoleCtrlHandler.restype = ctypes.wintypes.BOOL
kernel32.SetConsoleCtrlHandler.argtypes = [
    PHANDLER_ROUTINE,
    ctypes.wintypes.BOOL,
]


# =========================
# global state
# =========================

_main_thread_id = ctypes.wintypes.DWORD(0)
_exiting = threading.Event()


# =========================
# powercfg execution
# =========================

def run_powercfg(args: list[str]) -> None:
    subprocess.run(
        ["powercfg", *args],
        stdin=subprocess.DEVNULL,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
        creationflags=CREATE_NO_WINDOW,
        check=False,
    )


def apply_cpu_freq_limit() -> None:
    for scheme in SCHEMES:
        for setting in FREQ_SETTINGS:
            run_powercfg([
                "/setacvalueindex",
                scheme,
                "SUB_PROCESSOR",
                setting,
                str(ACMHz),
            ])

            run_powercfg([
                "/setdcvalueindex",
                scheme,
                "SUB_PROCESSOR",
                setting,
                str(DCMHz),
            ])

    run_powercfg([
        "/setactive",
        "SCHEME_CURRENT",
    ])


# =========================
# resume handling
# =========================

_apply_lock = threading.Lock()
_last_apply_time = 0.0


def apply_cpu_freq_limit_debounced() -> None:
    """
    PBT_APMRESUMEAUTOMATIC and PBT_APMRESUMESUSPEND
    may arrive consecutively, so suppress duplicate
    executions within a short interval.
    """
    global _last_apply_time

    if _exiting.is_set():
        return

    with _apply_lock:
        now = time.monotonic()

        if now - _last_apply_time < 5.0:
            return

        _last_apply_time = now
        apply_cpu_freq_limit()


def apply_async() -> None:
    if _exiting.is_set():
        return

    thread = threading.Thread(
        target=apply_cpu_freq_limit_debounced,
        daemon=True,
    )
    thread.start()


def power_notification_callback(context, event_type, setting) -> int:
    if event_type == PBT_APMRESUMESUSPEND:
        # Resume from sleep/hibernate initiated by user action
        apply_async()
        return 0

    if event_type == PBT_APMRESUMEAUTOMATIC:
        # Automatic resume.
        # Some environments only emit this event after hibernation.
        apply_async()
        return 0

    if event_type == PBT_APMSUSPEND:
        # Just before entering sleep/hibernate.
        # No action needed here.
        return 0

    return 0


# Keep a global reference so the callback is not garbage collected
_power_callback_ref = DEVICE_NOTIFY_CALLBACK_ROUTINE(power_notification_callback)
_notify_handle = ctypes.wintypes.HANDLE()


def register_power_notification() -> None:
    params = DEVICE_NOTIFY_SUBSCRIBE_PARAMETERS()
    params.Callback = _power_callback_ref
    params.Context = None

    result = powrprof.PowerRegisterSuspendResumeNotification(
        DEVICE_NOTIFY_CALLBACK,
        ctypes.byref(params),
        ctypes.byref(_notify_handle),
    )

    if result != 0:
        raise ctypes.WinError(result)


def unregister_power_notification() -> None:
    if _notify_handle:
        powrprof.PowerUnregisterSuspendResumeNotification(_notify_handle)


# =========================
# Ctrl+C handling
# =========================

def request_exit() -> None:
    _exiting.set()

    if _main_thread_id.value:
        user32.PostThreadMessageW(
            _main_thread_id.value,
            WM_QUIT,
            0,
            0,
        )


def console_ctrl_handler(ctrl_type: int) -> bool:
    if ctrl_type in (
        CTRL_C_EVENT,
        CTRL_BREAK_EVENT,
        CTRL_CLOSE_EVENT,
        CTRL_LOGOFF_EVENT,
        CTRL_SHUTDOWN_EVENT,
    ):
        request_exit()
        return True

    return False


# Keep a global reference so the handler is not garbage collected
_console_ctrl_handler_ref = PHANDLER_ROUTINE(console_ctrl_handler)


def register_console_ctrl_handler() -> None:
    ok = kernel32.SetConsoleCtrlHandler(
        _console_ctrl_handler_ref,
        True,
    )

    if not ok:
        raise ctypes.WinError(ctypes.get_last_error())


# =========================
# message loop
# =========================

def message_loop() -> None:
    msg = MSG()

    while not _exiting.is_set():
        ret = user32.GetMessageW(ctypes.byref(msg), None, 0, 0)

        if ret == 0:
            break

        if ret == -1:
            raise ctypes.WinError(ctypes.get_last_error())

        user32.TranslateMessage(ctypes.byref(msg))
        user32.DispatchMessageW(ctypes.byref(msg))


def main() -> None:
    global _main_thread_id

    _main_thread_id = ctypes.wintypes.DWORD(kernel32.GetCurrentThreadId())

    atexit.register(unregister_power_notification)

    register_console_ctrl_handler()

    # Apply once at startup
    apply_cpu_freq_limit_debounced()

    register_power_notification()

    try:
        message_loop()
    except KeyboardInterrupt:
        request_exit()
    finally:
        unregister_power_notification()


if __name__ == "__main__":
    main()
```

### What This Script Does

The script mainly does the following:

```text
Applies the CPU frequency limit once at startup
Registers for Windows suspend/resume notifications
Reapplies the powercfg settings after resuming from sleep or hibernation
Unregisters the notification handler before exiting, such as on Ctrl+C or shutdown
```

The resume-related events it watches are:

| Event                    | Meaning                                                                                  |
| ------------------------ | ---------------------------------------------------------------------------------------- |
| `PBT_APMRESUMESUSPEND`   | Resume from sleep or hibernation triggered by user action                                |
| `PBT_APMRESUMEAUTOMATIC` | Automatic resume. In some environments, only this event may be emitted after hibernation |
| `PBT_APMSUSPEND`         | Just before entering sleep or hibernation                                                |

`PBT_APMRESUMESUSPEND` and `PBT_APMRESUMEAUTOMATIC` may arrive consecutively, so the script suppresses duplicate executions within five seconds.

### How to Run It

Save the script as something like `apply_cpu_freq_limit_on_resume.py`, then run it from an elevated PowerShell window:

```powershell
python .\apply_cpu_freq_limit_on_resume.py
```

Changing `powercfg` settings requires administrator privileges, so you need to run it from PowerShell opened as administrator.

### Registering It at Startup

If you want to keep it running continuously, register it in Task Scheduler so that it starts at logon with administrator privileges.

Example settings:

```text
Trigger: At logon
Action: python.exe apply_cpu_freq_limit_on_resume.py
Privileges: Run with highest privileges
```

If `python.exe` is not in your PATH, specify the full path.

Example:

```text
C:\Users\<UserName>\AppData\Local\Programs\Python\Python312\python.exe
```

For the script argument, specify the full path to the saved `.py` file:

```text
C:\path\to\apply_cpu_freq_limit_on_resume.py
```

### Changing the Frequency Limit

To change the frequency limit, edit the following values near the top of the script:

```python
ACMHz = 1500
DCMHz = 1500
```

For example, to use 3000 MHz when plugged in and 2200 MHz on battery:

```python
ACMHz = 3000
DCMHz = 2200
```

To remove the limit, set both values to `0`, just like in the PowerShell version:

```python
ACMHz = 0
DCMHz = 0
```

### Notes

This script is Windows-only. It directly calls `powrprof.dll` and `user32.dll` through `ctypes`, so it will not work on macOS or Linux.

It also listens for resume events only while the script is running. If you want to use it continuously, registering it in Task Scheduler at logon is the practical approach.

## Disabling Boost Behavior as Well

If you want to suppress turbo-boost-like behavior in addition to limiting the maximum frequency, disable `PERFBOOSTMODE`.

```powershell
$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

foreach ($scheme in $Schemes) {
  powercfg /setacvalueindex $scheme SUB_PROCESSOR PERFBOOSTMODE 0 2>$null
  powercfg /setdcvalueindex $scheme SUB_PROCESSOR PERFBOOSTMODE 0 2>$null
}

powercfg /setactive SCHEME_CURRENT
```

`PERFBOOSTMODE 0` means that boost is disabled.

In my environment, however, the `PROCFREQMAX` settings alone were enough to produce the desired effect.

---

## Notes and Caveats

### 1. Run PowerShell as Administrator

Changing power settings with `powercfg` requires an elevated PowerShell window.

### 2. Do Not Forget to Define the Variables

One mistake I ran into was forgetting to define:

```powershell
$ACMHz
$DCMHz
```

If these variables are not defined, no value is passed to `powercfg`.

If `2>$null` is also used, the error output is hidden, making it harder to notice that the command failed.

During testing, it is better to start with an explicit command such as:

```powershell
powercfg /setacvalueindex OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX 1500
```

### 3. `PROCFREQMAX2` Varies by Environment

On my system, `PROCFREQMAX2` appeared in the output of `powercfg /qh`.

However, in some cases, running the following command may not show detailed information:

```powershell
powercfg /q OVERLAY_SCHEME_MAX SUB_PROCESSOR PROCFREQMAX2
```

For that reason, the script includes `PROCFREQMAX2` as a target while suppressing errors with `2>$null`.

### 4. Overlay Schemes May Show a Deprecation Warning

When running these commands, you may see a warning like this:

```text
Warning: Overlay schemes are deprecated and may not be supported in the future
```

In my environment, the warning appeared, but the settings still worked.

The same method may stop working in a future version of Windows.

---

## Summary

On Core Ultra-generation Windows PCs such as the CF-FV5USVCP, setting only `PROCFREQMAX` under `SCHEME_CURRENT` may not be enough to limit the maximum CPU frequency.

The key points that worked were:

```text
Configure overlay schemes as well as SCHEME_CURRENT
Configure PROCFREQMAX1 and PROCFREQMAX2 in addition to PROCFREQMAX
Configure both AC and DC values
Run powercfg /setactive SCHEME_CURRENT at the end
```

The final working script was:

```powershell
$ACMHz = 1500
$DCMHz = 1500

$Schemes = @(
  "SCHEME_CURRENT",
  "OVERLAY_SCHEME_MIN",
  "OVERLAY_SCHEME_MAX",
  "OVERLAY_SCHEME_HIGH"
)

$FreqSettings = @(
  "PROCFREQMAX",
  "PROCFREQMAX1",
  "PROCFREQMAX2"
)

foreach ($scheme in $Schemes) {
  foreach ($setting in $FreqSettings) {
    powercfg /setacvalueindex $scheme SUB_PROCESSOR $setting $ACMHz 2>$null
    powercfg /setdcvalueindex $scheme SUB_PROCESSOR $setting $DCMHz 2>$null
  }
}

powercfg /setactive SCHEME_CURRENT
```

If you want to reduce heat and fan noise on a CF-FV5USVCP, 1500–2200 MHz is a good range to try first. If you want to preserve more performance, adjust the limit upward to around 2500–3000 MHz.
