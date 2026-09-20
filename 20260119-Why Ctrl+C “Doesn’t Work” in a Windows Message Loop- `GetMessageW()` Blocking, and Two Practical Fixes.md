---
title: "Why Ctrl+C “Doesn’t Work” in a Windows Message Loop: `GetMessageW()` Blocking, and Two Practical Fixes"
description: "If you are running a Windows message loop from Python (for example via ctypes) and notice that..."
pubDatetime: 2026-01-19T08:08:07.545Z
---

If you are running a Windows message loop from Python (for example via `ctypes`) and notice that pressing **Ctrl+C** does not terminate the program immediately, you are not imagining things. In some setups, the process will only exit after the next Windows message arrives—such as a battery status update (`WM_POWERBROADCAST`)—which can make the script appear “stuck.”

This behavior has a clear cause: `GetMessageW()` can **block the thread completely**, and Python’s `KeyboardInterrupt` is typically raised only when control returns to the Python interpreter.

This article explains the mechanism and provides two field-ready mitigation patterns. The recommended approach is **Fix A**, which is immediate and robust.



## Root Cause: `GetMessageW()` Fully Blocks the Thread Until a Message Arrives

The underlying issue is that `GetMessageW()` **blocks the calling thread** until a message is placed into the thread’s message queue.

On Windows, Python’s Ctrl+C handling (SIGINT → `KeyboardInterrupt`) is, in practice, delivered as an exception when the interpreter regains execution—i.e., **when Python gets control back**.

So if your code is sitting inside `GetMessageW()` and *no message arrives*, then:

* `GetMessageW()` does not return,
* Python does not regain control,
* `KeyboardInterrupt` is not raised,
* and your `except KeyboardInterrupt:` block never runs.

That is exactly why the program may only terminate once “the next message” (battery change, timer tick, etc.) occurs.



## Two Practical Fixes (Recommended: Fix A)

Below are two pragmatic countermeasures:

* **Fix A (Recommended):** Convert Ctrl+C into a Windows *event* and wait on *both* the event and the message queue using `MsgWaitForMultipleObjectsEx()`. This gives immediate shutdown.
* **Fix B (Simpler):** Use `WM_TIMER` to periodically “wake” `GetMessageW()` so Ctrl+C is processed on the next tick. Minimal changes, but not immediate.



# Fix A (Recommended): Turn Ctrl+C Into an Event and Wait with `MsgWaitForMultipleObjectsEx()` (Immediate Exit)

### High-level design

1. Install a Windows **Console Control Handler** to catch Ctrl+C (and related console events).
2. Signal a manual-reset event created via `CreateEventW()` when Ctrl+C is received.
3. Replace the blocking `GetMessageW()` loop with `MsgWaitForMultipleObjectsEx()` to wait for either:

   * (1) the stop event, or
   * (2) input in the message queue

This eliminates the “blocked forever in `GetMessageW()`” failure mode. When Ctrl+C is pressed, the event is set and the wait returns immediately, allowing you to exit the loop deterministically.



## Additional constants and API prototypes (add near your “Function prototypes” section)

```python
#  Immediate Ctrl+C shutdown support 
PM_REMOVE = 0x0001
QS_ALLINPUT = 0x04FF
MWMO_INPUTAVAILABLE = 0x0004

WAIT_OBJECT_0 = 0x00000000
WAIT_FAILED   = 0xFFFFFFFF
INFINITE      = 0xFFFFFFFF

CTRL_C_EVENT        = 0
CTRL_BREAK_EVENT    = 1
CTRL_CLOSE_EVENT    = 2
CTRL_LOGOFF_EVENT   = 5
CTRL_SHUTDOWN_EVENT = 6

# kernel32: event + console handler
kernel32.CreateEventW.argtypes = [wintypes.LPVOID, wintypes.BOOL, wintypes.BOOL, wintypes.LPCWSTR]
kernel32.CreateEventW.restype  = wintypes.HANDLE

kernel32.SetEvent.argtypes = [wintypes.HANDLE]
kernel32.SetEvent.restype  = wintypes.BOOL

kernel32.CloseHandle.argtypes = [wintypes.HANDLE]
kernel32.CloseHandle.restype  = wintypes.BOOL

ConsoleCtrlHandler = ctypes.WINFUNCTYPE(wintypes.BOOL, wintypes.DWORD)
kernel32.SetConsoleCtrlHandler.argtypes = [ConsoleCtrlHandler, wintypes.BOOL]
kernel32.SetConsoleCtrlHandler.restype  = wintypes.BOOL

# user32: wait + peek
user32.MsgWaitForMultipleObjectsEx.argtypes = [
    wintypes.DWORD, ctypes.POINTER(wintypes.HANDLE), wintypes.DWORD,
    wintypes.DWORD, wintypes.DWORD
]
user32.MsgWaitForMultipleObjectsEx.restype = wintypes.DWORD

user32.PeekMessageW.argtypes = [ctypes.POINTER(MSG), wintypes.HWND, wintypes.UINT, wintypes.UINT, wintypes.UINT]
user32.PeekMessageW.restype  = wintypes.BOOL
```



## Replace the message-loop section in `main()` (swap out your `# 4) message loop`)

Replace your current `GetMessageW()` loop with the following (the rest of your script can typically remain unchanged):

```python
        print("Listening for battery percentage changes. Press Ctrl+C to exit.")

        # - Prepare Ctrl+C stop event -
        stop_event = kernel32.CreateEventW(None, True, False, None)
        if not stop_event:
            raise ctypes.WinError(ctypes.get_last_error())

        ctrl_handler = None
        try:
            @ConsoleCtrlHandler
            def ctrl_handler(ctrl_type):
                if ctrl_type in (
                    CTRL_C_EVENT, CTRL_BREAK_EVENT, CTRL_CLOSE_EVENT,
                    CTRL_LOGOFF_EVENT, CTRL_SHUTDOWN_EVENT
                ):
                    # Signal stop event (immediately breaks the wait)
                    kernel32.SetEvent(stop_event)
                    # True: mark as handled and prevent Python's default Ctrl+C handling
                    return True
                return False

            if not kernel32.SetConsoleCtrlHandler(ctrl_handler, True):
                raise ctypes.WinError(ctypes.get_last_error())

            handles = (wintypes.HANDLE * 1)(stop_event)

            # - Wait for either messages or stop_event -
            msg = MSG()
            while True:
                r = user32.MsgWaitForMultipleObjectsEx(
                    1, handles, INFINITE, QS_ALLINPUT, MWMO_INPUTAVAILABLE
                )

                if r == WAIT_OBJECT_0:
                    # stop_event signaled -> exit immediately
                    break

                if r == WAIT_OBJECT_0 + 1:
                    # Messages available -> drain the queue
                    while user32.PeekMessageW(ctypes.byref(msg), None, 0, 0, PM_REMOVE):
                        if msg.message == WM_QUIT:
                            return 0
                        user32.TranslateMessage(ctypes.byref(msg))
                        user32.DispatchMessageW(ctypes.byref(msg))
                    continue

                if r == WAIT_FAILED:
                    raise ctypes.WinError(ctypes.get_last_error())

            return 0

        finally:
            # Unregister console handler and close event handle
            if ctrl_handler:
                kernel32.SetConsoleCtrlHandler(ctrl_handler, False)
            if stop_event:
                kernel32.CloseHandle(stop_event)
```



## Why this works (key points)

* Ctrl+C sets a Windows **event** immediately.
* `MsgWaitForMultipleObjectsEx()` returns as soon as either:

  * the stop event is signaled, or
  * there are messages in the queue
* You no longer rely on Python’s `KeyboardInterrupt` being raised at the “right time.”
* This removes the structural dependency on “the next Windows message” to regain control.

In other words: the message loop becomes interruptible in a first-class Windows-native way.



# Fix B (Simple): Use `WM_TIMER` to Periodically Wake `GetMessageW()` (Minimal Changes)

If you want minimal code churn, another approach is to ensure `GetMessageW()` returns regularly by generating a periodic message.

Using `SetTimer()` to emit `WM_TIMER` at a fixed interval ensures that:

* `GetMessageW()` will return at least every N milliseconds,
* and after you press Ctrl+C, Python will regain control on the next timer tick.

This allows the script to exit, but with a delay bounded by the timer interval. It is an effective “good enough” mitigation for some environments, but it is not truly immediate and is conceptually less clean than Fix A.
