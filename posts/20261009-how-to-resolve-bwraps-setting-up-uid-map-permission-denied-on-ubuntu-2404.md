---
title: "How to Resolve \"setting up uid map: Permission denied\" in bwrap on Ubuntu 24.04"
description: "This article explains how to resolve the error by applying an AppArmor profile specifically for bwrap, while maintaining system-wide restrictions."
pubDatetime: 2026-10-09T02:30:00+09:00
---

When attempting to build a sandboxed environment using Bubblewrap (`bwrap`) on Ubuntu 24.04, the following error occurred:

```text
bwrap: setting up uid map: Permission denied
```

After investigation, it was determined that the AppArmor User Namespace restrictions introduced in Ubuntu 24.04 were the cause.

In this case, the issue was resolved by applying an AppArmor profile specifically for bwrap, without disabling AppArmor's security restrictions system-wide.

This article will introduce the cause of the error, the investigation process, and the steps taken to resolve it.

## Environment

- OS: Ubuntu 24.04 LTS
- Execution Environment: Virtual Machine
- Tools Used: Bubblewrap (bwrap)
- Security Mechanism: AppArmor
- Error Encountered: `bwrap: setting up uid map: Permission denied`

## 1. The Problem

Bubblewrap is a tool that uses Linux Namespaces and other features to run processes in isolated environments.

In this case, bwrap was used to run the Codex CLI in an isolated environment.

However, when attempting to start the sandbox, the following error occurred, causing it to fail:

```text
bwrap: setting up uid map: Permission denied
```

`uid map` is a mechanism for setting the correspondence between user IDs within a User Namespace and user IDs on the host side.

This error indicates that bwrap was unable to set up the UID mapping.

## 2. The Cause: AppArmor Restrictions in Ubuntu 24.04

In Ubuntu 24.04, AppArmor applies additional restrictions to the use of User Namespaces by non-privileged users.

Therefore, even if the User Namespace is enabled on the Linux kernel side, AppArmor profiles may reject bwrap's operation.

Upon examining the kernel logs, AppArmor denials were recorded.

### Checking AppArmor Logs

The following command extracts logs related to the error:

```bash
sudo journalctl -k -b --no-pager |
  grep -Ei 'apparmor="DENIED"|uid_map|bwrap'
```

The following is a sample of the logs that were found:

```text
apparmor="AUDIT"
operation="userns_create"
profile="unconfined"
comm="bwrap"
target="unprivileged_userns"
```

```text
apparmor="DENIED"
operation="capable"
profile="unprivileged_userns"
comm="bwrap"
capname="setpcap"
```

```text
apparmor="DENIED"
operation="open"
profile="unprivileged_userns"
name="proc/5296/uid_map"
requested_mask="wr"
denied_mask="wr"
```

The important part is the denial of access to `uid_map`.

bwrap attempted to open the file required to set up UID mapping, but AppArmor denied read and write access.

Therefore, the issue was not a simple file permission problem, but rather a restriction imposed by AppArmor's security policy.

## 3. Solution: Apply an AppArmor Profile Specifically for bwrap

The problem was resolved by following these steps:

### Step 1: Install Necessary Packages

First, update the package list and install AppArmor-related packages:

```bash
sudo apt update
sudo apt install apparmor-profiles apparmor-utils
```

`apparmor-profiles` provides additional AppArmor profiles.

`apparmor-utils` provides management tools for configuring and checking AppArmor status.

### Step 2: Place the bwrap Profile

Next, copy the AppArmor profile for bwrap to `/etc/apparmor.d/`:

```bash
sudo install -m 0644 \
  /usr/share/apparmor/extra-profiles/bwrap-userns-restrict \
  /etc/apparmor.d/bwrap-userns-restrict
```

This command performs the following:

- Uses the bwrap profile provided in `/usr/share/apparmor/extra-profiles/`.
- Places it in `/etc/apparmor.d/` to manage it.
- Sets the file permissions to `0644` using `-m 0644`.

The key point is to use a dedicated profile for bwrap instead of disabling AppArmor's User Namespace restrictions system-wide.

※ Before executing, make sure there is no existing bwrap profile in the destination directory. `install` will overwrite files with the same name.

### Step 3: Load the AppArmor Profile

Load the placed profile into AppArmor:

```bash
sudo apparmor_parser -r \
  /etc/apparmor.d/bwrap-userns-restrict
```

`apparmor_parser` is a command for parsing AppArmor profiles and loading them into the kernel.

The `-r` option replaces and loads an existing profile.

This enables the AppArmor profile for bwrap.

## 4. Verification

After configuration, verify the basic operation of bwrap:

```bash
bwrap \
  --ro-bind / / \
  --dev /dev \
  --proc /proc \
  --unshare-user \
  --unshare-pid \
  -- /bin/sh -c 'id; echo "bwrap OK"'
```

If it works correctly, the following will be displayed:

```text
uid=1000(...) gid=1000(...) groups=...
bwrap OK
```

The display of UID, GID, etc. will vary depending on the execution environment.

In this environment, the introduction of the AppArmor profile eliminated the `setting up uid map: Permission denied` error in the original bwrap command.

## 5. Why Not Disable User Namespace Restrictions?

On the internet, the following command is often introduced as a solution to this problem:

```bash
sudo sysctl -w kernel.apparmor_restrict_unprivileged_userns=0
```

This setting disables AppArmor's additional restrictions for non-privileged User Namespaces system-wide.

Indeed, bwrap may work with this method.

However, it is not necessary to relax system-wide security restrictions just to get bwrap to work.

As shown in this case, a dedicated AppArmor profile for bwrap can be applied, allowing the issue to be resolved while maintaining system-wide restrictions.

Note that applying a dedicated profile also involves granting permissions to bwrap, which does not completely eliminate the risk. It is important to check the contents of the profile being used and grant permissions only to the extent necessary.

## 6. Summary

If you encounter the following error on Ubuntu 24.04:

```text
bwrap: setting up uid map: Permission denied
```

AppArmor's User Namespace restrictions may be the cause.

First, check the kernel logs to see if denials are occurring for `unprivileged_userns` or `uid_map`.

In this environment, the problem was resolved with the following commands:

```bash
sudo apt update
sudo apt install apparmor-profiles apparmor-utils

sudo install -m 0644 \
  /usr/share/apparmor/extra-profiles/bwrap-userns-restrict \
  /etc/apparmor.d/bwrap-userns-restrict

sudo apparmor_parser -r \
  /etc/apparmor.d/bwrap-userns-restrict
```

**The key is to apply a dedicated AppArmor profile for bwrap instead of disabling AppArmor's restrictions system-wide.**

Hopefully, this will be helpful for those who are experiencing the same error when using Bubblewrap on Ubuntu 24.04.
