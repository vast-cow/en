---
title: "How to Use a Tool to Remove a Specific App from Windows “File Associations”"
description: "On Windows, uninstalled applications or portable applications can sometimes leave behind only their..."
pubDatetime: 2026-08-18T08:41:19.711Z
---

On Windows, uninstalled applications or portable applications can sometimes leave behind only their file-association information in the registry.

For example, when you display “Open with” for files such as `.epub` or `.pdf`, an application you no longer use may remain listed as an option.

This section explains how to use a Python tool that examines file associations registered under `HKEY_CLASSES_ROOT` and deletes registry keys associated with a specified executable file.

## What This Tool Does

This tool examines the keys directly under `HKEY_CLASSES_ROOT`, abbreviated as `HKCR`.

For each key, it checks:

```text
HKEY_CLASSES_ROOT\<key name>\shell\open\command
```

and retrieves the executable filename from the command set as the default value.

For example, suppose the following entry exists:

```text
"C:\Program Files\Calibre2\ebook-viewer.exe" "%1"
```

If the specified executable filename is `ebook-viewer.exe`, this association is detected as a target.

When run normally, the tool deletes the target registry key. If you add `--dry-run`, it only displays the targets without deleting them.

## Requirements

This script is for Windows only.

It uses only the following Python standard-library modules:

* `argparse`
* `ctypes`
* `os`
* `winreg`

Therefore, no additional packages need to be installed.

It can be used on a Windows system with Python 3 installed.

## First, Save the Script

Save the code under a filename such as:

```text
remove_association.py
```

Open Command Prompt or PowerShell and move to the directory where you saved the file.

Example:

```powershell
cd C:\Tools
```

## Always Check with `--dry-run` First

Because this tool deletes registry keys, it is safer not to delete anything immediately. First, use `--dry-run` to review the targets.

The syntax is:

```powershell
python remove_association.py <executable filename> --dry-run
```

For example, to check entries associated with `calibre.exe`, run:

```powershell
python remove_association.py calibre.exe --dry-run
```

If targets are found, the output will look like this:

```text
Found 2 association(s).

HKEY_CLASSES_ROOT\Calibre...
  command = "C:\Program Files\Calibre2\calibre.exe" "%1"

HKEY_CLASSES_ROOT\...
  command = "C:\Program Files\Calibre2\calibre.exe" "%1"

dry-run: No keys were deleted.
```

If the final line says:

```text
dry-run: No keys were deleted.
```

then the registry has not been modified.

## Actually Deleting the Entries

After reviewing the `--dry-run` results and confirming that only unnecessary associations were detected, run the command again without `--dry-run`.

```powershell
python remove_association.py calibre.exe
```

If deletion succeeds, the tool displays:

```text
DELETED: HKEY_CLASSES_ROOT\...
```

If multiple associations are found, the detected keys are deleted one by one.

## Specify Only the Executable Filename

As a general rule, specify only the executable filename as the argument, not the full path.

For example, instead of:

```text
C:\Program Files\Calibre2\calibre.exe
```

specify:

```text
calibre.exe
```

The tool parses the command registered in the registry according to Windows command-line rules, extracts only the filename of the executable at the beginning of the command, and compares that filename.

The comparison is case-insensitive. Therefore:

```text
CALIBRE.EXE
```

and:

```text
calibre.exe
```

are treated as the same filename.

## Why It Does Not Use a Simple String Search

An association command does not necessarily contain only the executable path.

For example, it may look like:

```text
"C:\Program Files\Example\viewer.exe" "%1"
```

or:

```text
"C:\Program Files\Example\viewer.exe" --open "%1"
```

In addition, Windows command lines have their own rules for handling quotation marks and spaces.

For this reason, the tool parses the command string using the Windows API function:

```text
CommandLineToArgvW
```

It then extracts the executable filename from the first parsed argument using:

```python
os.path.basename(parts[0])
```

This structure is less prone to false positives than a simple substring check such as:

```python
if "calibre.exe" in command:
```

## Support for `REG_EXPAND_SZ`

The tool also supports commands stored not only as ordinary `REG_SZ` strings, but also as `REG_EXPAND_SZ` values containing environment variables.

For example:

```text
"%ProgramFiles%\Example\viewer.exe" "%1"
```

In this case, the tool expands the environment variables using:

```python
os.path.expandvars(value)
```

before parsing the command.

## How Associations Are Searched

The core of the search process is `find_associations()`.

```python
def find_associations(target_exe: str):
```

This function sequentially enumerates the keys directly under `HKEY_CLASSES_ROOT` using `winreg.EnumKey()`.

For each key, it reads:

```text
<key>\shell\open\command
```

and, if the executable filename matches the specified name, returns it as a deletion candidate with:

```python
yield key_name, command
```

In other words, this tool searches **the `shell\open\command` entry under each key directly beneath HKCR**.

Note that it is not a tool that comprehensively searches every type of file-association information that may exist in Windows.

## Deletion Includes Subkeys

A registry key cannot be deleted directly if it contains subkeys.

For that reason, `delete_registry_tree()` recursively deletes child keys before deleting the parent key.

Conceptually, if the structure looks like this:

```text
Target key
├─ DefaultIcon
├─ shell
│  └─ open
│     └─ command
└─ other entries
```

the tool deletes entries from the bottom of the hierarchy upward, then finally deletes the “Target key” itself.

Therefore, when this tool deletes a detected key, it removes not only `shell\open\command`, but **the entire association key**.

This is an important point.

## If You Get a Permission Error

During deletion, you may see:

```text
ACCESS DENIED: HKEY_CLASSES_ROOT\...
```

This can occur when the current user does not have permission to modify the registry key.

If necessary, run the command from PowerShell or Command Prompt opened with “Run as administrator.”

However, a permission error does not mean you should automatically delete the key with administrator privileges.

First, review the `--dry-run` output and confirm that the key is truly unnecessary.

## If No Target Is Found

If no matching association exists, the tool displays a message such as:

```text
No associations matching 'calibre.exe' were found.
```

In this case, nothing is deleted.

However, there may also be cases where the association is visible in Windows but is not detected by this tool.

Windows stores file-association information in multiple locations, and this tool checks only registrations in the following form:

```text
HKEY_CLASSES_ROOT\<key>\shell\open\command
```

## Usage Example

To only check associations linked to `foo.exe`, run:

```powershell
python remove_association.py foo.exe --dry-run
```

After reviewing the results, to delete them, run:

```powershell
python remove_association.py foo.exe
```

For example, suppose the detected result is:

```text
Found 1 association(s).

HKEY_CLASSES_ROOT\Foo.Document
  command = "C:\OldApps\Foo\foo.exe" "%1"
```

If you run the tool normally in this state, the deletion target is not simply:

```text
HKEY_CLASSES_ROOT\Foo.Document\shell\open\command
```

Instead, the entire tree under:

```text
HKEY_CLASSES_ROOT\Foo.Document
```

is deleted.

Therefore, if `Foo.Document` contains other registered information, that information will also be lost.

## Precautions

This tool directly deletes registry keys. It does not provide a function to restore deleted keys.

One especially important point is that when it finds a `shell\open\command` matching the specified EXE name, it recursively deletes not just that `command` key, but **the association key itself directly under HKCR**.

For that reason, the following workflow is recommended:

1. Confirm the target EXE filename.
2. Search using `--dry-run`.
3. Review every registry key that is displayed.
4. If necessary, export the target key from Registry Editor to create a backup.
5. Only after confirming there is no problem, run the tool normally.

It is particularly advisable to avoid targeting built-in Windows applications or applications that are currently in use.

## Summary

This tool examines Windows `HKEY_CLASSES_ROOT`, searches for associations where the specified executable is registered under:

```text
<key>\shell\open\command
```

and deletes those keys.

The basic usage is simple.

To only check:

```powershell
python remove_association.py calibre.exe --dry-run
```

To actually delete:

```powershell
python remove_association.py calibre.exe
```

Because the tool directly modifies the registry, **the most important step is to review the targets with `--dry-run` first.**

It can be useful for cleaning up old file associations left behind after uninstalling applications or associations for applications you no longer need, but because it deletes the entire association key, review the contents carefully and use it with caution.

```python
import argparse
import ctypes
import os
import winreg
from ctypes import wintypes


shell32 = ctypes.WinDLL("shell32", use_last_error=True)
kernel32 = ctypes.WinDLL("kernel32", use_last_error=True)

shell32.CommandLineToArgvW.argtypes = [
    wintypes.LPCWSTR,
    ctypes.POINTER(ctypes.c_int),
]
shell32.CommandLineToArgvW.restype = ctypes.POINTER(wintypes.LPWSTR)

kernel32.LocalFree.argtypes = [wintypes.HLOCAL]
kernel32.LocalFree.restype = wintypes.HLOCAL


def split_windows_command(command: str) -> list[str]:
    argc = ctypes.c_int()

    argv = shell32.CommandLineToArgvW(command, ctypes.byref(argc))
    if not argv:
        raise ctypes.WinError(ctypes.get_last_error())

    parts = [argv[i] for i in range(argc.value)]
    kernel32.LocalFree(argv)

    return parts


def get_open_command(root, key_name: str) -> str | None:
    subkey = rf"{key_name}\shell\open\command"

    try:
        with winreg.OpenKey(root, subkey, 0, winreg.KEY_READ) as key:
            value, value_type = winreg.QueryValueEx(key, "")

            if value_type in (winreg.REG_SZ, winreg.REG_EXPAND_SZ):
                if value_type == winreg.REG_EXPAND_SZ:
                    value = os.path.expandvars(value)
                return value

    except (FileNotFoundError, PermissionError, OSError):
        pass

    return None


def executable_name_from_command(command: str) -> str | None:
    try:
        parts = split_windows_command(command)
    except (ValueError, OSError):
        return None

    if not parts:
        return None

    return os.path.basename(parts[0])


def find_associations(target_exe: str):
    target_exe = target_exe.casefold()

    root = winreg.HKEY_CLASSES_ROOT
    index = 0

    while True:
        try:
            key_name = winreg.EnumKey(root, index)
        except OSError:
            break

        index += 1

        command = get_open_command(root, key_name)
        if command is None:
            continue

        executable_name = executable_name_from_command(command)
        if executable_name is None:
            continue

        if executable_name.casefold() == target_exe:
            yield key_name, command


def delete_registry_tree(root, subkey: str):
    try:
        with winreg.OpenKey(
            root,
            subkey,
            0,
            winreg.KEY_READ | winreg.KEY_WRITE,
        ) as key:
            while True:
                try:
                    child = winreg.EnumKey(key, 0)
                except OSError:
                    break

                delete_registry_tree(root, rf"{subkey}\{child}")

        winreg.DeleteKey(root, subkey)

    except FileNotFoundError:
        pass


def main():
    parser = argparse.ArgumentParser(
        description=(
            r"Checks the executable name in HKCR\<key>\shell\open\command and "
            "deletes keys associated with the specified exe."
        )
    )

    parser.add_argument(
        "exe",
        help="Executable filename to search for. Example: calibre.exe",
    )

    parser.add_argument(
        "--dry-run",
        action="store_true",
        help="List only the target keys without deleting them",
    )

    args = parser.parse_args()

    matches = list(find_associations(args.exe))

    if not matches:
        print(f"No associations matching {args.exe!r} were found.")
        return

    print(f"Found {len(matches)} association(s).")
    print()

    for key_name, command in matches:
        print(fr"HKEY_CLASSES_ROOT\{key_name}")
        print(f"  command = {command}")

    if args.dry_run:
        print()
        print("dry-run: No keys were deleted.")
        return

    print()
    print("Deleting.")

    for key_name, command in matches:
        full_name = fr"HKEY_CLASSES_ROOT\{key_name}"

        try:
            delete_registry_tree(winreg.HKEY_CLASSES_ROOT, key_name)
            print(f"DELETED: {full_name}")
        except PermissionError:
            print(f"ACCESS DENIED: {full_name}")
        except OSError as e:
            print(f"ERROR: {full_name}: {e}")


if __name__ == "__main__":
    main()
```
