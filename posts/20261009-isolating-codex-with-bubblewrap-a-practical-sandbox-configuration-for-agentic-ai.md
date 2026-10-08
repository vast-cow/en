---
title: "Isolating Codex with Bubblewrap: A Practical Sandbox Configuration for Agent AI"
description: "This article introduces a configuration that uses Bubblewrap to make the file system read-only, allowing writes only in necessary locations. By separating Codex's Exec Server and Client, the configuration enables free operation within a project while preventing unintended modifications to the home directory or system settings."
pubDatetime: 2026-10-09T00:10:00+09:00
---

Giving an AI agent the ability to execute shell commands significantly expands its capabilities.

It can edit files, run programs, install packages, build, and test. While this autonomy is convenient, it also introduces the risk of unintentionally damaging the host environment.

However, setting up a dedicated VM or container environment each time you use an agent AI can be cumbersome. You also want it to have access to the tools and libraries you normally use.

Therefore, I considered a configuration that uses Linux's **Bubblewrap (bwrap)** to isolate and run Codex.

This configuration isn't foolproof in terms of security. Nevertheless, I believe it strikes a good balance between practicality and safety for the agent AI use cases I envision.

## What are we trying to prevent?

The goal here isn't complete isolation from malicious attackers.

Instead, we want to prevent the agent AI from unintentionally modifying files outside its working directory or breaking the development environment you normally use by executing unintended commands.

For example, we want to reduce the risk of the agent accidentally deleting files in the home directory or rewriting system settings.

At the same time, we want to allow it to freely edit, build, and test code within the project.

Therefore, the approach taken here is simple:

**The host file system is, in principle, read-only, and only the necessary locations are writable.**

Furthermore, to separate Codex itself from the command execution environment, we use Codex's `exec-server`.

## Overall Configuration

This configuration uses two Bubblewrap environments.

```text
Host Linux
│
├── Bubblewrap ①
│   └── Codex Exec Server
│       ├── Host entire system: Read Only (in principle)
│       ├── Project: Read / Write
│       ├── HOME: Temporary area
│       └── WebSocket :8765
│
└── Bubblewrap ②
    └── Codex Client
        ├── Host entire system: Read Only (in principle)
        ├── Working directory: Mount output/
        ├── ~/.codex: Read / Write
        └── Connect to Exec Server
```

The Codex Client is responsible for interacting with the AI and providing execution instructions, while the Exec Server is responsible for executing commands.

The two communicate via WebSocket.

The important point is that **the Client and Exec Server have different views of the file system and different write permissions.**

The Client sees `output/` as its working directory. The Exec Server, on the other hand, is allowed to read and write to the entire original project.

Therefore, the files visible to the Client are not the only ones the agent can operate on. The permissions of the Exec Server apply when the actual commands are executed.

This is a design choice I'm intentionally allowing.

## Implementation

This assumes that you have `bwrap` and the Codex CLI installed in your Linux environment.

Also, assume that the Codex executable is located at `~/.codex/packages/standalone/current/bin/codex`. Using the `exec-server` requires a compatible version of Codex.

First, create an output directory within the project.

```bash
mkdir -p output
```

### ① Launching the Exec Server

In the first Bubblewrap, launch the Codex Exec Server, which is dedicated to command execution.

```bash
bwrap \
  --die-with-parent \
  --unshare-pid \
  --unshare-ipc \
  --unshare-uts \
  \
  --ro-bind / / \
  --dev /dev \
  --proc /proc \
  \
  --perms 0700 \
  --tmpfs "$HOME" \
  --ro-bind /etc/skel/.profile "$HOME/.profile" \
  --ro-bind /etc/skel/.bashrc "$HOME/.bashrc" \
  --ro-bind /etc/skel/.bash_logout "$HOME/.bash_logout" \
  \
  --dir "$HOME/.codex" \
  --ro-bind "$HOME/.codex/packages" "$HOME/.codex/packages" \
  \
  --dir "$HOME/.local" \
  --dir "$HOME/.local/bin" \
  --symlink \
    "$HOME/.codex/packages/standalone/current/bin/codex" \
    "$HOME/.local/bin/codex" \
  \
  --bind "$PWD" "$PWD" \
  \
  --tmpfs /tmp \
  --chmod 1777 /tmp \
  --tmpfs /var/tmp \
  --chmod 1777 /var/tmp \
  \
  --clearenv \
  --setenv HOME "$HOME" \
  --setenv PATH "$HOME/.local/bin:/usr/local/cuda/bin:/usr/local/bin:/usr/bin:/bin" \
  --setenv TERM "${TERM:-xterm-256color}" \
  --setenv LANG "${LANG:-C.UTF-8}" \
  \
  --chdir "$PWD" \
  codex \
  --sandbox danger-full-access \
  --ask-for-approval never \
  exec-server \
  --listen ws://127.0.0.1:8765
```

This configuration has several key points.

#### Host file system is read-only

```bash
--ro-bind / /
```

The host's root file system is exposed in read-only mode.

This allows you to continue using commands and libraries already installed on the host.

The advantage is that you don't need to build a separate root file system for the isolated environment.

However, this doesn't hide the host's files. Files that can be read by the OS's access permissions are generally visible.

#### Replace the home directory with a temporary area

```bash
--perms 0700
--tmpfs "$HOME"
```

The home directory is replaced with a `tmpfs`.

This means that the configuration files and authentication information in the original home directory are not visible.

The shell initialization files are read from `/etc/skel` in read-only mode.

Also, only the packages needed by Codex are exposed.

```bash
--dir "$HOME/.codex"
--ro-bind "$HOME/.codex/packages" "$HOME/.codex/packages"
```

This is done because the Exec Server doesn't need Codex's authentication information or the entire user configuration.

#### Allow writes only to the project

```bash
--bind "$PWD" "$PWD"
```

This is the most important setting for the Exec Server.

The current project directory is mounted in read-write mode on top of the file system that was made read-only.

This allows the agent to edit files within the project.

Conversely, writes to normal host files outside the project are restricted via the file system.

Of course, explicitly created writable areas such as `/tmp` and a temporary HOME are exceptions.

#### Separate temporary directories

```bash
--tmpfs /tmp
--chmod 1777 /tmp

--tmpfs /var/tmp
--chmod 1777 /var/tmp
```

Temporary directories used for builds and tests are also separated.

Because the host's `/tmp` is not shared, normal temporary file operations do not affect the host.

#### Initialize environment variables

```bash
--clearenv
```

Host environment variables are not inherited, and only the necessary ones are explicitly set.

Because environment variables can contain API keys and tokens, this design avoids passing unnecessary information.

Note that proxy and various development tools also require environment variables, so you will need to add them individually if needed.

### ② Launching the Codex Client

Next, launch the Codex Client from a separate terminal.

```bash
bwrap \
  --die-with-parent \
  --unshare-pid \
  --unshare-ipc \
  --unshare-uts \
  \
  --ro-bind / / \
  --dev /dev \
  --proc /proc \
  \
  --perms 0700 \
  --tmpfs "$HOME" \
  --ro-bind /etc/skel/.profile "$HOME/.profile" \
  --ro-bind /etc/skel/.bashrc "$HOME/.bashrc" \
  --ro-bind /etc/skel/.bash_logout "$HOME/.bash_logout" \
  \
  --bind "$HOME/.codex" "$HOME/.codex" \
  \
  --dir "$HOME/.local" \
  --dir "$HOME/.local/bin" \
  --symlink \
    "$HOME/.codex/packages/standalone/current/bin/codex" \
    "$HOME/.local/bin/codex" \
  \
  --bind "$PWD/output" "$PWD" \
  \
  --tmpfs /tmp \
  --chmod 1777 /tmp \
  --tmpfs /var/tmp \
  --chmod 1777 /var/tmp \
  \
  --clearenv \
  --setenv HOME "$HOME" \
  --setenv PATH "$HOME/.local/bin:/usr/local/cuda/bin:/usr/local/bin:/usr/bin:/bin" \
  --setenv TERM "${TERM:-xterm-256color}" \
  --setenv LANG "${LANG:-C.UTF-8}" \
  \
  --chdir "$PWD" \
  --setenv CODEX_EXEC_SERVER_URL ws://127.0.0.1:8765 \
  codex \
  --sandbox danger-full-access
```

The Client's configuration is similar to the Exec Server, but there are two important differences.

#### Share the Codex configuration

```bash
--bind "$HOME/.codex" "$HOME/.codex"
```

The Client needs Codex's authentication information and settings, so `~/.codex` is mounted in read-write mode.

Unlike the Exec Server, this exposes the authentication information.

Therefore, this configuration does not completely protect authentication information.

#### Replace the working directory with output

```bash
--bind "$PWD/output" "$PWD"
```

The Client sees `output/` as the current working directory.

For example, suppose the host has the following structure:

```text
project/
├── src/
├── README.md
├── package.json
└── output/
```

On the Client side, the host's `output/` is mounted at the `project/` location.

On the other hand, the Exec Server has read and write access to the entire original `project/`.

Therefore, the files visible to the Client are not the only ones the agent can operate on. The permissions of the Exec Server apply when the commands are actually executed.

This is an intentional design choice.

**Making the Client's workspace smaller is different from limiting the command execution permissions.**

In this configuration, I chose a design that allows access to the entire project.

### ③ Connect to the Exec Server

Finally, specify the following environment variable:

```bash
--setenv CODEX_EXEC_SERVER_URL ws://127.0.0.1:8765
```

This tells Codex to use the external Exec Server as the execution environment.

Because the Client and Exec Server use the same host's network namespace, they can communicate using the loopback address.

I intentionally didn't specify `--unshare-net` here.

I wanted to maintain the network connection for Codex to communicate with the API while also making it easy to communicate with the Exec Server.

## Is `danger-full-access` okay?

In this configuration, I'm using the following setting for Codex:

```bash
--sandbox danger-full-access
```

It sounds dangerous, but I'm intentionally using it.

Instead of layering additional sandbox controls on top of Codex, I'm taking the approach of managing access permissions to the file system at the OS level with Bubblewrap.

Therefore, I'm not imposing strong restrictions on the Codex side, but rather allowing it to execute the commands it needs.

However, `danger-full-access` itself is not a safe setting. The permissions for files, networks, and sockets exposed by Bubblewrap must be considered separately.

Also, in the original example, I'm specifying `--sandbox` and `--ask-for-approval` when launching the Exec Server, but you shouldn't assume that the actual permissions of the Exec Server are controlled solely by these CLI options.

Also, if you want to explicitly allow operation without approval, specify `--ask-for-approval never` on the Client side as well.

The core protection mechanism in this configuration is the Bubblewrap settings.

## Security Trade-offs

This configuration is not a silver bullet.

In particular, you need to understand the following points before using it:

| Risk | Mitigation in this configuration |
|---|---|
| Normal writes to files outside the project | Read-only in principle |
| Accidental operations on the home directory | Isolated with tmpfs |
| Impact on the host environment due to temporary files | Isolated with tmpfs |
| Destruction of files within the project | Allowed |
| Access to potentially sensitive information readable on the host | Not completely prevented |
| Information leakage via network | Not prevented |
| Unauthorized connections to the Exec Server from the same host | Not prevented due to lack of authentication |

In particular, be aware that `--ro-bind / /` does not completely hide information from the host.

Files that can be read by the OS's access permissions are generally visible, and network communication is also allowed, so this configuration does not prevent information leakage.

Also, be careful about accessing host services via Unix Domain Sockets that exist in `/run`, etc. A read-only file system does not necessarily mean that operations on the service are prohibited.

Furthermore, the WebSocket for the Exec Server is listening on `127.0.0.1` in this example, but in this example, no authentication is configured.

It is not directly exposed to the external network, but it may be possible to connect from other processes on the same host.

If you want a more robust security boundary, you should consider WebSocket authentication, reducing the number of directories exposed, blocking Unix Domain Sockets, and separating the Network Namespace.

## Why Choose This Configuration?

Reading this far, you might think that the security measures are insufficient.

In fact, it is not sufficient as a general-purpose sandbox for safely executing potentially malicious code.

However, what I have in mind is the use case where I want to protect my normal development environment from unintended file operations by the agent by running Codex in a trusted local environment.

I'm assuming that I'm going to let the agent edit files within the project.

At worst, the project might be destroyed. However, I believe that this can be handled with Git or backups.

On the other hand, I want to avoid changing my home directory or system settings.

For this purpose, Bubblewrap is very easy to use.

No need to maintain container images, and you can directly access the tools and libraries you normally use. Compared to Docker or VMs, it has less overhead in terms of operational burden for my use case.

Of course, this is not a generalization of "sufficient security."

If you are dealing with sensitive information or running untrusted code provided by a third party, you should choose a more rigorous isolation method.

## Summary

When you think about security measures for agent AIs, it's easy to end up with a complex configuration.

Network Namespace separation, seccomp, AppArmor, SELinux, containers, VMs, and so on – there are countless measures you can add.

However, security measures are not simply about being as strict as possible.

If the restrictions are too strong, they may impair the usability of the AI agent, or if the maintenance and management of the environment become too burdensome, it may deviate from the original purpose.

In this configuration, I combined Bubblewrap and the Codex Exec Server to prevent the agent from freely modifying the entire host while allowing it to operate freely within the project.

There are still risks that cannot be prevented, and it is not guaranteed as a robust security boundary.

Nevertheless, I believe that this simple configuration is just right for the agent AI use cases I envision.

**What is important is not to eliminate all risks, but to clearly define what needs to be protected and what can be tolerated, and then to establish the necessary boundaries.**

---

### References

- [Bubblewrap — GitHub](https://github.com/containers/bubblewrap)
- [Bubblewrap — bwrap(1) Manual](https://manpages.debian.org/trixie/bubblewrap/bwrap.1.en.html)
- [Codex Exec Server — README](https://github.com/openai/codex/blob/main/codex-rs/exec-server/README.md)
- [Codex — Exec Serverを使ってTUIを起動するサンプル](https://github.com/openai/codex/blob/main/scripts/run_tui_with_exec_server.sh)

*This article describes a configuration example and does not represent the results of security verification in a real environment. Codex's `exec-server` is an experimental feature, and the configuration and behavior may change depending on the version.*
