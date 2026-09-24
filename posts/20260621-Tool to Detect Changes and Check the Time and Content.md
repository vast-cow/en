---
title: "Tool to Detect Changes and Check the Time and Content"
description: "Monitor a command's standard output with a Python script that timestamps changes and runs a trigger command at startup and whenever the output changes."
pubDatetime: 2026-06-21T06:24:56.387Z
updatedDate: 2026-06-21T07:16:23.457Z
---

This script is a tool that periodically checks the output of a specified command and displays the time and the new content whenever the output changes.

It can also execute another command when a change is detected.

For example, it can be used to monitor file contents, system status, command results, and similar outputs. When a change occurs, it can log the event, send notifications, or trigger additional processing.

## Purpose

The primary purpose of this tool is to make it easy to observe changes in a command's standard output.

Normally, repeatedly running a command manually to check for changes can be tedious. This script automates that process by executing the command at a specified interval and comparing the current output with the previous one.

When the output changes, it displays:

* The time the change occurred
* The updated standard output

In addition, because it can execute another command when a change is detected, it can be used not only for monitoring but also for triggering follow-up actions.

## Basic Behavior

This script uses two commands.

The first is the command to monitor. It is executed periodically, and its standard output is compared with the previous result.

The second is the trigger command. It is executed once at startup and then again each time the monitored output changes.

When the script starts, it first runs the monitored command and displays its output as the initial state. It then executes the trigger command once.

After that, the monitored command is executed at the specified interval. If its standard output differs from the previous output, the script displays the current time and the new content, then executes the trigger command.

## Usage

Save the script to a file and make it executable.

```bash
chmod +x watch_change.py
```

The basic usage format is:

```bash
./watch_change.py -c "command_to_monitor" -t "command_to_run_on_change"
```

For example, to monitor a file and display a message whenever it changes:

```bash
./watch_change.py -c "cat sample.txt" -t "echo changed"
```

In this example, the script periodically checks the contents of `sample.txt`. When the file changes, it displays the time and the updated content, then runs `echo changed`.

## Specifying the Monitoring Interval

By default, the monitored command is executed once per second.

To change the interval, use `-i` or `--interval`.

```bash
./watch_change.py -c "cat sample.txt" -t "echo changed" -i 5
```

In this example, the contents of `sample.txt` are checked every five seconds.

Use a shorter interval if you need more frequent updates, or a longer interval to reduce system load.

## Displaying Trigger Command Output

By default, the standard output of the trigger command is not displayed. This keeps the monitoring output easier to read.

If you want the trigger command's output to be shown as well, add `--show-trigger-stdout`.

```bash
./watch_change.py \
  -c "cat sample.txt" \
  -t "echo changed" \
  --show-trigger-stdout
```

With this option enabled, the output of the trigger command will be displayed whenever a change is detected.

## Output Format

When the script starts, it displays the initial standard output.

```text
[2026-06-21T10:00:00] initial stdout
current content
```

When a change is detected during monitoring, the output looks like this:

```text
[2026-06-21T10:00:05] stdout changed
updated content
```

This allows you to see both when the change occurred and what the new content is.

## Errors and Warnings

If either the monitored command or the trigger command does not exit successfully, a warning is displayed on standard error.

Examples include situations where:

* The command does not exist
* A target file cannot be found

However, the script does not terminate immediately when such warnings occur. Monitoring continues, making it suitable for situations where temporary errors are expected.

## Stopping the Script

To stop the script, press `Ctrl+C` in the terminal.

When stopped, the following message is displayed:

```text
stopped
```

## Examples

### Monitor a File

```bash
./watch_change.py -c "cat status.txt" -t "echo status changed"
```

Each time the contents of `status.txt` change, the updated content and the time of the change are displayed.

### Monitor Command Output

```bash
./watch_change.py -c "date +%M" -t "echo minute changed"
```

In this example, the output changes whenever the minute changes. It serves as a simple demonstration for testing the script.

### Log Changes

```bash
./watch_change.py \
  -c "cat status.txt" \
  -t "echo changed at $(date) >> change.log"
```

This records the time of each detected change in a log file.

## Notes

This script compares only the standard output of the monitored command. Standard error output is not included in the comparison.

Also, command strings are executed as commands split by whitespace rather than through a shell. If you need more advanced shell features, it is often easier to use `sh -c`.

```bash
./watch_change.py \
  -c "sh -c 'ls -l *.txt'" \
  -t "sh -c 'echo changed >> change.log'"
```

## Summary

This script is a simple tool for monitoring command output and displaying both the time and content whenever a change occurs.

It can be used for tasks such as monitoring file states, tracking command results, logging changes, and triggering notifications or other automated actions. It requires no complicated configuration—simply specify the command to monitor and the command to run when a change is detected.

```python
#!/usr/bin/env python3
import argparse
import shlex
import subprocess
import sys
import time
from datetime import datetime


DEFAULT_INTERVAL_SECONDS = 1.0


def now() -> str:
    return datetime.now().isoformat(timespec="seconds")


def print_stdout(data: bytes) -> None:
    sys.stdout.buffer.write(data)
    if data and not data.endswith(b"\n"):
        print()
    sys.stdout.flush()


def run_capture(cmd: list[str]) -> subprocess.CompletedProcess[bytes]:
    return subprocess.run(
        cmd,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        check=False,
    )


def run_trigger(
    cmd: list[str],
    *,
    show_stdout: bool = False,
) -> subprocess.CompletedProcess[bytes]:
    return subprocess.run(
        cmd,
        stdout=None if show_stdout else subprocess.DEVNULL,
        stderr=None,
        check=False,
    )


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description=(
            "Run cmd2 initially, watch stdout of cmd1, "
            "and run cmd2 whenever stdout changes."
        )
    )

    parser.add_argument(
        "-c",
        "--command",
        required=True,
        help="command to monitor periodically",
    )

    parser.add_argument(
        "-t",
        "--trigger",
        required=True,
        help="command to run initially and whenever stdout changes",
    )

    parser.add_argument(
        "-i",
        "--interval",
        type=float,
        default=DEFAULT_INTERVAL_SECONDS,
        help=f"polling interval in seconds; default: {DEFAULT_INTERVAL_SECONDS}",
    )

    parser.add_argument(
        "--show-trigger-stdout",
        action="store_true",
        help="show stdout from cmd2; default is to suppress it",
    )

    return parser.parse_args()


def main() -> None:
    args = parse_args()

    watch_cmd = shlex.split(args.command)
    trigger_cmd = shlex.split(args.trigger)

    if not watch_cmd:
        print("error: empty watch command", file=sys.stderr)
        sys.exit(2)

    if not trigger_cmd:
        print("error: empty trigger command", file=sys.stderr)
        sys.exit(2)

    if args.interval <= 0:
        print("error: interval must be positive", file=sys.stderr)
        sys.exit(2)

    try:
        initial_time = now()
        initial_result = run_capture(watch_cmd)
        prev = initial_result.stdout

        print(f"[{initial_time}] initial stdout")
        print_stdout(prev)

        if initial_result.returncode != 0:
            print(
                f"[{now()}] warning: initial command exited with "
                f"{initial_result.returncode}",
                file=sys.stderr,
            )

        initial_trigger_result = run_trigger(
            trigger_cmd,
            show_stdout=args.show_trigger_stdout,
        )
        if initial_trigger_result.returncode != 0:
            print(
                f"[{now()}] warning: initial trigger exited with "
                f"{initial_trigger_result.returncode}",
                file=sys.stderr,
            )

        while True:
            time.sleep(args.interval)

            current_result = run_capture(watch_cmd)
            current = current_result.stdout

            if current_result.returncode != 0:
                print(
                    f"[{now()}] warning: command exited with "
                    f"{current_result.returncode}",
                    file=sys.stderr,
                )

            if current != prev:
                print(f"[{now()}] stdout changed")
                print_stdout(current)

                trigger_result = run_trigger(
                    trigger_cmd,
                    show_stdout=args.show_trigger_stdout,
                )
                if trigger_result.returncode != 0:
                    print(
                        f"[{now()}] warning: trigger exited with "
                        f"{trigger_result.returncode}",
                        file=sys.stderr,
                    )

                prev = current

    except KeyboardInterrupt:
        print("\nstopped", file=sys.stderr)
        sys.exit(130)


if __name__ == "__main__":
    main()
```
