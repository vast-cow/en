---
title: "Running a non-root Samba server on Singularity (Apptainer) using a non-privileged port and sharing `$HOME`"
description: "This guide documents how to run a Samba server inside a Singularity / Apptainer container under the..."
pubDatetime: 2026-04-16T12:56:45.549Z
---

This guide documents how to run a **Samba server inside a Singularity / Apptainer container** under the following constraints:

* No root privileges on the host
* Use of a **non-privileged port** (e.g., 1445 instead of 445)
* Sharing the user’s **home directory (`$HOME`)**
* All writable paths confined to user space

This setup is particularly useful in **HPC or restricted environments**.

---

## Design Principles

* Run `smbd` as a **non-root user**
* Use **port ≥1024** (e.g., `1445`)
* Disable NetBIOS (`nmbd` not used)
* Use `[homes]` for per-user home directory sharing
* Redirect all writable paths to `$HOME`
* Use **bind mounts** for filesystem access

---

## 1. Build the Container Image

### Definition file (`samba.def`)

```def
Bootstrap: docker
From: ubuntu:24.04

%post
    export DEBIAN_FRONTEND=noninteractive
    apt-get update
    apt-get install -y samba smbclient
    apt-get clean
    rm -rf /var/lib/apt/lists/*
```

### Build

```bash
singularity build --fakeroot samba.sif samba.def
```

---

## 2. Prepare Host Directories

```bash
mkdir -p ~/samba/{etc,log,lock,run,cache,private}
chmod 700 ~/samba/private
mkdir -p ~/samba/run/ncalrpc
```

---

## 3. Create `smb.conf`

`~/samba/etc/smb.conf`:

```ini
[global]
   server role = standalone server
   workgroup = WORKGROUP
   netbios name = MYSMB

   security = user
   map to guest = never

   smb ports = 1445
   disable netbios = yes

   lock directory = /hosthome/USER/samba/lock
   pid directory = /hosthome/USER/samba/run
   state directory = /hosthome/USER/samba/cache
   cache directory = /hosthome/USER/samba/cache
   private dir = /hosthome/USER/samba/private

   log file = /hosthome/USER/samba/log/log.%m
   max log size = 1000

   ncalrpc dir = /hosthome/USER/samba/run/ncalrpc

   load printers = no
   printing = bsd
   printcap name = /dev/null
   disable spoolss = yes

[homes]
   browseable = no
   read only = no
   valid users = %S
   create mask = 0600
   directory mask = 0700
```

Replace `USER`:

```bash
sed -i "s|USER|$USER|g" ~/samba/etc/smb.conf
```

---

## 4. Bind Mount Setup

Bind your home directory into the container:

```bash
export SMB_BIND="$HOME:/hosthome/$USER"
```

---

## 5. Set Samba Password (with fakeroot)

```bash
singularity exec --fakeroot \
  --bind "$SMB_BIND" \
  samba.sif \
  smbpasswd -c /hosthome/$USER/samba/etc/smb.conf -a root
```

In this setup, the password is assigned to the `root` user inside Samba.

---

## 6. Run the Server (Important Fixes Applied)

### Prepare log directory bind

```bash
mkdir -p ~/samba/varlog
```

### Run

```bash
singularity exec --fakeroot \
  --bind "$SMB_BIND" \
  --bind "$HOME/samba/varlog:/var/log/samba" \
  samba.sif \
  smbd --foreground --no-process-group --debug-stdout \
    -s /hosthome/$USER/samba/etc/smb.conf \
    -p 1445
```

---

## 7. Verify

```bash
ss -ltnp | grep 1445
```

```bash
smbclient -L //127.0.0.1 -p 1445 -U root
```

---

## Common Pitfalls and Fixes

### 1. Read-only `/var/log/samba`

**Error:**

```plaintext
Unable to open new log file '/var/log/samba/log.smbd'
```

**Fix:**

```bash
--bind "$HOME/samba/varlog:/var/log/samba"
```

---

### 2. Missing `/run/samba/ncalrpc`

**Error:**

```plaintext
Failed to create pipe directory /run/samba/ncalrpc
```

**Fix:**

Add to `smb.conf`:

```ini
ncalrpc dir = /hosthome/$USER/samba/run/ncalrpc
```

Create directory:

```bash
mkdir -p ~/samba/run/ncalrpc
```

---

### 3. Invalid `-S` option

**Error:**

```plaintext
Invalid option -S
```

**Fix:**

```bash
smbd --foreground --no-process-group
```

---

### 4. Privileged ports not allowed

* Ports `445` / `139` require root
* Use `1445` or another port ≥1024

---

## Summary

This setup enables:

* Running Samba **without root privileges**
* Fully contained execution in Singularity
* Direct sharing of `$HOME`
* Compatibility with restricted environments (e.g., HPC)

### Key Takeaways

1. Redirect all writable paths to user space
2. Override container defaults (`/var/log`, `/run`)
3. Always use a non-privileged port

---

## Conceptual Note

This approach is less about “containerizing Samba” and more about:

> Running Samba entirely in user space, with Singularity acting as the runtime environment.
