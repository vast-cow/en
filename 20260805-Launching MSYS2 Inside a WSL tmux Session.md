---
title: "Launching MSYS2 Inside a WSL tmux Session"
description: "This setup allows you to start a WSL tmux session from MSYS2 and then launch an interactive MSYS2..."
pubDatetime: 2026-08-05T07:47:59.918Z
---

This setup allows you to start a WSL `tmux` session from MSYS2 and then launch an interactive MSYS2 shell inside that `tmux` session. It combines the convenience of running `tmux` in WSL with the familiar MSYS2 environment.

## Purpose

The main goal is to keep `tmux` running in WSL while using MSYS2 as the interactive shell inside each `tmux` pane or window. This approach lets you manage terminal sessions with WSL `tmux` while continuing to work with MSYS2 tools.

The launcher script starts WSL, connects to a dedicated `tmux` server, and passes the required environment variables so that MSYS2 starts correctly.

## Launcher Script

The launcher script performs the following tasks:

* Exports the environment variables required by both WSL and MSYS2.
* Prevents automatic path conversion where it is not desired.
* Starts `tmux` inside the target WSL distribution.
* Uses a dedicated `tmux` socket and configuration file.

With this script, you can launch or attach to the `tmux` session directly from MSYS2.

## tmux Configuration

The `tmux` configuration changes the default command so that every new pane starts an MSYS2 shell instead of a Linux shell.

Before launching MSYS2, the configuration clears the `PWD` environment variable and updates `WSLENV` so that the necessary environment variables are available inside the MSYS2 process. It then starts an interactive MSYS2 Bash session.

The configuration also adds `MSYSTEM` and `WSLENV` to `tmux`'s `update-environment` list. This ensures that newly created panes inherit the correct environment.

## How to Use

1. Save the launcher script and make it executable.
2. Place the `tmux` configuration in the specified configuration file.
3. Run the launcher script from MSYS2.
4. A WSL `tmux` session will start, and each new pane or window will automatically open an MSYS2 shell.

This setup provides a simple way to use WSL `tmux` as the session manager while working in the MSYS2 environment.

Please write a simple article **in English** about the content below labeled “---- content ----”.

* Write the title as a level-1 heading using `#`.
* You may use `##`, `###`, and `####` to denote sections, subsections, and sub-subsections.
* Do not include any questions asking for additional suggestions.
* Focus on the purpose and how to use it, and keep technical explanations to a minimum.


wrapper executed on msys2:
```bash
#!/usr/bin/env bash
set -euo pipefail

export WSLENV="MSYSTEM${WSLENV:+:$WSLENV}"
export MSYS2_ARG_CONV_EXCL='*'
export MSYS2_ENV_CONV_EXCL="WSLENV;TMUX;TMUX_PANE${MSYS2_ENV_CONV_EXCL:+;$MSYS2_ENV_CONV_EXCL}"

exec wsl.exe -d tmux --exec tmux \
    -L msys2 \
    -f /path/to/tmux-msys2.conf \
    "$@"
```

tmux-msys2.conf inside WSL:
```conf
set-option -g default-command '
unset PWD
WSLENV="MSYSTEM:PWD/p:$WSLENV"
exec /mnt/c/msys64/usr/bin/bash.exe --login -i
'
set-option -ag update-environment MSYSTEM
set-option -ag update-environment WSLENV
```
