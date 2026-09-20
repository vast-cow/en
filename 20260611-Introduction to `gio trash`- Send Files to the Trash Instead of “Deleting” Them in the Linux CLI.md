---
title: "Introduction to `gio trash`: Send Files to the Trash Instead of “Deleting” Them in the Linux CLI"
description: "When deleting files from the Linux command line, many people instinctively use rm. However, rm..."
pubDatetime: 2026-06-11T11:31:24.786Z
---

When deleting files from the Linux command line, many people instinctively use `rm`. However, `rm` generally performs a **permanent deletion without using the Trash**, making it unforgiving of mistakes.

In GNOME/GLib-based environments, you can use `gio trash` to **move files and directories to the Trash**, just like a GUI file manager does. `gio trash` is a subcommand of `gio` and also supports listing Trash contents, restoring items, and emptying the Trash. Ubuntu and Arch Linux man pages document options such as `gio trash --list`, `--restore`, and `--empty`. ([Ubuntu Manpages][1])

---

## What Is `gio trash`?

`gio` is a command-line utility provided by GLib/GIO. It can operate not only on local files but also on URIs and virtual file systems supported by GIO.

The `trash` subcommand is `gio trash`.

```bash
gio trash FILE...
```

Instead of immediately deleting a file like `rm FILE`, it moves the target to the Trash.

This is similar to right-clicking a file in GNOME Files (formerly Nautilus) and selecting **Move to Trash**. GVfs is the user-space virtual filesystem implementation for GIO and provides backends for Trash, SFTP, SMB, WebDAV, and more. ([wiki.gnome.org][2])

---

## Basic Usage

### Send a File to the Trash

```bash
gio trash memo.txt
```

You can specify multiple files:

```bash
gio trash a.txt b.txt c.txt
```

Directories are also supported:

```bash
gio trash old-project/
```

When possible, metadata is stored so the item can later be restored to its original location.

---

## View the Contents of the Trash

```bash
gio trash --list
```

You can also list the `trash://` URI directly:

```bash
gio list trash://
```

The Arch Linux man page documents both `gio trash --list` and `gio list trash://` as ways to inspect the Trash. ([Arch Manual Pages][3])

Example:

```text
trash:///memo.txt    /home/user/Documents/memo.txt
trash:///old-project /home/user/work/old-project
```

The exact output format may vary by environment and version, but it generally shows the item's Trash URI and original location.

---

## Restore Files from the Trash

Use `--restore` to restore an item:

```bash
gio trash --restore trash:///memo.txt
```

The important detail is that restoration expects a `trash://` URI, not a normal filesystem path.

The Ubuntu man page explains that `--restore` restores an item to its original location and expects a `trash://` URI. If the original directory no longer exists, it will be recreated. ([Ubuntu Manpages][1])

A typical workflow looks like this:

```bash
gio trash --list
gio trash --restore trash:///memo.txt
```

---

## Empty the Trash

To permanently empty the Trash:

```bash
gio trash --empty
```

Use caution. While `gio trash` is useful for safer deletion, `--empty` is effectively the final deletion step.

---

## Ignore Missing Files

Use `-f` or `--force` to ignore files that do not exist or cannot be trashed:

```bash
gio trash -f maybe-exists.txt
```

The Arch Linux man page describes `-f, --force` as an option that ignores nonexistent files and files that cannot be moved to the Trash. ([Arch Manual Pages][3])

---

## Difference Between `rm` and `gio trash`

| Command             | Behavior                    | Recoverable       |
| ------------------- | --------------------------- | ----------------- |
| `rm file`           | Deletes the file            | Usually difficult |
| `gio trash file`    | Moves the file to the Trash | Yes               |
| `gio trash --empty` | Empties the Trash           | Usually difficult |

`rm` is powerful for scripts and temporary-file cleanup, but it can be risky for manual operations. In situations like the following, `gio trash` is often the safer choice:

```bash
gio trash ~/Downloads/*
gio trash old-report.pdf
gio trash tmp-output/
```

That said, `gio trash` is only a Trash mechanism—it is not a substitute for backups.

---

## Relationship to `gvfs-trash`

Older articles sometimes reference the `gvfs-trash` command:

```bash
gvfs-trash file.txt
```

Today, `gio trash` is generally the preferred approach.

Ubuntu-related documentation and discussions often note that `gvfs-trash` is deprecated and recommend using `gio trash` instead. ([Ask Ubuntu][4])

For new setups, use `gio trash` rather than `gvfs-trash`.

---

## Difference Between `gio trash` and `trash-cli`

Another popular Trash utility for Linux is `trash-cli`:

```bash
trash-put file.txt
trash-list
trash-restore
trash-empty
```

A practical comparison:

| Use Case                                                     | Recommended Tool |
| ------------------------------------------------------------ | ---------------- |
| Use standard GNOME/GLib functionality                        | `gio trash`      |
| Want a desktop-environment-independent CLI tool              | `trash-cli`      |
| Prefer interactive restoration workflows                     | `trash-cli`      |
| Want something lightweight that already exists on the system | `gio trash`      |

On GNOME desktops or systems with GLib/GVfs installed, `gio trash` is often available without additional packages. On servers or minimal installations, it may not be installed.

---

## Shell Alias Examples

To make it easier to use instead of `rm`, create aliases:

```bash
alias del='gio trash'
alias trash='gio trash'
```

Add them to `~/.bashrc` or `~/.zshrc`:

```bash
echo "alias del='gio trash'" >> ~/.bashrc
source ~/.bashrc
```

Then use:

```bash
del old.txt
del old-directory/
```

It is technically possible to replace `rm` itself:

```bash
alias rm='gio trash'
```

However, this is generally not recommended. It can break assumptions in scripts, system administration tasks, and shared environments. A separate alias such as `del` or `trash` is usually the safer choice.

---

## Caveats

### 1. Not All Filesystems Behave the Same Way

The Trash location can vary depending on where the file resides.

Red Hat documentation explains that files under a user's home directory typically use `$XDG_DATA_HOME/Trash`, but Trash locations may differ depending on the filesystem, and not all filesystems support the Trash concept. ([Red Hat Documentation][5])

### 2. Avoid `sudo gio trash`

Running `gio trash` as root may cause files to be processed using the root user's environment and Trash location rather than your own.

For normal files, use:

```bash
gio trash file.txt
```

If you truly need to remove root-owned files, administrators often prefer a clearly defined deletion policy and tools such as `sudo rm` rather than a Trash-based workflow.

### 3. Restoring Requires a `trash://` URI

Commands like this may fail:

```bash
gio trash --restore /home/user/Documents/memo.txt
```

Instead, identify the item with `gio trash --list` and restore it using its `trash://` URI:

```bash
gio trash --restore trash:///memo.txt
```

---

## Practical Examples

### Move Unwanted ZIP Files in Downloads to the Trash

```bash
gio trash ~/Downloads/*.zip
```

### Move All `.log` Files in the Current Directory to the Trash

```bash
gio trash ./*.log
```

### Review the Trash Before Emptying It

```bash
gio trash --list
gio trash --empty
```

### Configure Deletion Aliases

```bash
cat >> ~/.bashrc <<'EOF'
alias del='gio trash'
alias trash='gio trash'
EOF

source ~/.bashrc
```

---

## Summary

`gio trash` is a practical command for safely sending files to the Trash from the Linux command line.

```bash
gio trash file.txt
gio trash --list
gio trash --restore trash:///file.txt
gio trash --empty
```

`rm` performs immediate deletion, whereas `gio trash` moves items to the Trash. Simply understanding and using that distinction can significantly reduce the risk of accidental data loss during CLI work.

For day-to-day manual file management, `gio trash` is often the better choice. Use `rm` when you explicitly intend permanent deletion.

[1]: https://manpages.ubuntu.com/manpages/jammy/man1/gio.1.html?utm_source=chatgpt.com "gio - GIO commandline tool"
[2]: https://wiki.gnome.org/Projects/gvfs?utm_source=chatgpt.com "Projects/gvfs"
[3]: https://man.archlinux.org/man/gio.1?utm_source=chatgpt.com "gio(1) - Arch manual pages"
[4]: https://askubuntu.com/questions/213533/command-to-move-a-file-to-trash-via-terminal?utm_source=chatgpt.com "Command to move a file to Trash via Terminal"
[5]: https://docs.redhat.com/ko/documentation/red_hat_enterprise_linux/9/html/administering_the_system_using_the_gnome_desktop_environment/available-gio-commands_managing-storage-volumes-in-gnome?utm_source=chatgpt.com "12.6. Available GIO Commands | GNOME Desktop Environment Administration"
