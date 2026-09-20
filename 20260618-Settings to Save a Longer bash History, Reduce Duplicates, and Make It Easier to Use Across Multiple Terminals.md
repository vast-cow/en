---
title: "Settings to Save a Longer bash History, Reduce Duplicates, and Make It Easier to Use Across Multiple Terminals"
description: "If you use bash regularly, you often want to find commands you ran earlier. With the default..."
pubDatetime: 2026-06-18T08:55:33.669Z
updatedDate: 2026-06-26T05:45:36.310Z
---

If you use bash regularly, you often want to find commands you ran earlier. With the default settings, the number of saved history entries may be small, or history may not be shared well when you work with multiple terminals open.

This article introduces basic settings for saving bash history for a longer period and making it easier to handle across multiple sessions.

## Settings

Add the following settings to `~/.bashrc`.

```bash
export HISTSIZE=100000
export HISTFILESIZE=200000
shopt -s histappend
export PROMPT_COMMAND='history -a; history -c; history -r'
export HISTCONTROL=ignoredups:erasedups:ignorespace
```

After adding the settings, apply them with the following command.

```bash
source ~/.bashrc
```

## Purpose

The main purpose of these settings is to make bash command history more convenient to use.

By increasing `HISTSIZE` and `HISTFILESIZE`, you can keep commands you ran in the past for longer. This makes it easier to reuse frequently used commands or complex commands you ran a little while ago.

Also, enabling `histappend` makes bash append to the history file instead of overwriting it. Even when you are working with multiple terminals open, your history is less likely to disappear.

In addition, the `PROMPT_COMMAND` setting saves history again each time you run a command and also reloads history executed in other terminals. This makes it easier to share history across multiple sessions.

## Meaning of Each Setting

### `HISTSIZE`

```bash
export HISTSIZE=100000
```

Specifies the number of history entries kept in the current shell.

In this example, up to 100,000 history entries are kept in memory.

### `HISTFILESIZE`

```bash
export HISTFILESIZE=200000
```

Specifies the number of entries saved in the history file.

In this example, up to 200,000 history entries are saved in the history file.

### `histappend`

```bash
shopt -s histappend
```

This setting appends to the history file when bash exits instead of overwriting it.

When using multiple terminals, this helps reduce problems such as only the history from the terminal closed last being preserved.

### `PROMPT_COMMAND`

```bash
export PROMPT_COMMAND='history -a; history -c; history -r'
```

Specifies commands to run each time the prompt is displayed.

With this setting, the following processes are performed in order.

#### `history -a`

Appends the history added in the current session to the history file.

#### `history -c`

Clears the history in the current shell once.

#### `history -r`

Reloads history from the history file.

This makes commands executed in other terminals more likely to be reflected in the current terminal’s history.

## Usage

After configuring these settings, just use bash as usual.

To search past commands, press `Ctrl + r` and use history search. Enter part of a command to search for commands you ran in the past.

```bash
Ctrl + r
```

You can also display the history list with the `history` command.

```bash
history
```

Since the history becomes long, it is useful to combine it with `grep`.

```bash
history | grep ssh
history | grep docker
history | grep rsync
```

## Notes

Because these settings save history for a long time, you need to be careful not to enter confidential information such as passwords or tokens directly on the command line.

If you accidentally enter confidential information, you need to delete the relevant line from the history file. Normally, the bash history file is located here.

```bash
~/.bash_history
```

## Summary

With these settings in place, you can save bash history for a long period and more easily share history when working with multiple terminals.

If you often use the terminal, this can improve work efficiency by allowing you to quickly reuse past commands.
