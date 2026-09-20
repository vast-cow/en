---
title: "Designing a Secure Directory Layout for Services That Start as Root and Then Drop Privileges"
description: "Separate root-owned configuration and code from writable service state using FHS directories, restrictive permissions, and systemd filesystem controls after dropping privileges."
pubDatetime: 2026-01-19T11:12:07.163Z
---

Services that **start as root**, perform a small set of privileged operations, and then **drop privileges and run long-term as an unprivileged user** can be made significantly more robust by enforcing a clear rule:

**Strictly separate “paths writable by root” from “paths writable by the post-drop service user,” and minimize writable locations after privilege drop.**

A practical way to achieve this is to follow the **Filesystem Hierarchy Standard (FHS)** and design your file placement so that only a small, explicitly intended set of directories remains writable after the service becomes unprivileged.



## Recommended FHS-Aligned Directory Design

### 1) Read-Mostly Configuration (Managed by Root)

**`/etc/<svc>/`**

Use this for **static configuration** files that are read at startup (YAML/TOML/INI, etc.), not modified during runtime.

Typical permissions:

* General configuration: `0644 root:root` or `0640 root:<svcgroup>`
* **Secrets (API keys, DB passwords, private keys):**
  `0640 root:<svcgroup>` or `0600 root:root` (depending on how the secret is passed to the process)

Key rule: **Do not place files under `/etc` that the service updates while running.**
Runtime writes to `/etc` blur the privilege boundary, invite configuration corruption, and increase operational risk.

**Additional note (why this matters operationally):**
If the service user can write anything under `/etc/<svc>`, then a compromise after the privilege drop can become a *persistent* compromise (e.g., rewriting config to execute a different binary, change endpoints, or disable auth). Treat `/etc` as **root-owned policy**, not mutable state.



### 1b) Service Code, Libraries, and Helper Binaries (Root-Owned, Read-Only in Practice)

**`/usr/lib/<svc>/` or `/usr/libexec/<svc>/`**

Use this for:

* Executables
* Root-only helper tools (only if truly necessary)

These directories should be **read-only in practice** (writable only by root).

**Added clarification: `/usr/lib/<svc>` vs `/usr/libexec/<svc>`**
Both are commonly used for “service-owned implementation artifacts,” but they have different intent:

* **`/usr/lib/<svc>/`** is typically used for **libraries and service-private modules** (plugins, `.so` files, language runtime packages, internal assets). It is “lib-like” in the sense that content is usually *loaded* rather than invoked directly by users.
* **`/usr/libexec/<svc>/`** is typically used for **service-private helper executables**—programs intended to be invoked by the service (or by the service manager), not by interactive users and not via `PATH`.

In practice, distributions vary, and you will see either directory used for both roles. The key security property is consistent: **root-owned and not writable by the post-drop user**.

**Security note (supply-chain and persistence):**
If the post-drop user can write into `/usr` locations, then an attacker can replace binaries/modules and gain persistent code execution on restart. Keeping `/usr` root-owned and read-only is a core part of maintaining a trustworthy runtime.



### 2) Runtime-Only (Volatile) State

**`/run/<svc>/`** (historically `/var/run`)

Use this for:

* UNIX domain sockets
* PID files
* short-lived state
* locks

Typical permissions:

* `0750 root:<svcgroup>` or `0750 <svcuser>:<svcgroup>`

Because `/run` is cleared on reboot, **never store persistent data here**.

**Added guidance (ownership patterns):**

* If the service manager (e.g., systemd) creates the directory and the service runs unprivileged, prefer:

  * directory owner: `<svcuser>:<svcgroup>`
  * mode: `0750`
* If root needs to create certain files (rare), keep root as owner and use group write sparingly, with explicit group membership.

**Safety note (socket/lock handling):**
UNIX sockets and lockfiles can become privilege boundary choke points. Ensure:

* directory is not world-writable
* avoid predictable filenames in shared temp dirs
* set restrictive `umask` early



### 3) Persistent Service Data (Writable After Privilege Drop)

**`/var/lib/<svc>/`**

Use this for:

* Databases
* queues
* indexes
* persistent caches that are expensive to rebuild

Typical ownership/mode:

* Owner: `<svcuser>:<svcgroup>`
* Mode: `0750` (or similar)

Design principle: treat this as **the primary (and ideally only) “core” writable area** after privileges are dropped.

**Added guidance (state integrity):**
Persistent state is frequently the highest-value target after compromise. Consider:

* using integrity checks or signed metadata if practical
* separating subdirectories by trust level (e.g., `db/`, `uploads/`, `tokens/`)
* ensuring backups do not silently reintroduce compromised state



### 4) Cache (Rebuildable, Can Be Deleted)

**`/var/cache/<svc>/`**

Use this for:

* rebuildable caches
* downloaded artifacts
* longer-lived temporary data

Typical ownership/mode:

* Owner: `<svcuser>`
* Mode: `0750`

**Added guidance (cache poisoning):**
Caches are often fed by external inputs. If cache contents influence execution (templates, bytecode, plugins, dynamically loaded assets), treat them closer to **code** than **data**. In that case, consider placing them under root-managed directories or using verification (hash/signature) before use.



### 5) Logs

**`/var/log/<svc>/`** (or prefer journald)

Best practice is to **centralize logs in journald** and reduce file-based logging.

If writing to files:

* Assume **logrotate**
* Ensure the service can append safely according to your logging strategy

Typical ownership/mode (varies with approach):

* `0750 root:<svcgroup>` (adjust based on whether the service user writes directly, or logging is handled via syslog/journald)

**Added guidance (log permissions and rotation):**
Two common safe patterns:

1. **Service writes to stdout/stderr; journald collects**

   * minimal filesystem writable surface
   * avoids logrotate permission complications

2. **Service writes to files**

   * ensure the service only needs append, not arbitrary rewrite
   * coordinate with logrotate using `copytruncate` or `create` with correct ownership (depending on your application’s reopen behavior)
   * avoid letting the service user write logs in a directory that also contains root-written logs unless permissions are carefully designed



### 6) Temporary Files

* Short-lived: **`/tmp`** (must be handled safely: `mkstemp`, `O_EXCL`, avoid fixed filenames)
* Longer-lived: **`/var/tmp`**
* Safer pattern: create a **service-specific temporary directory** (e.g., `/run/<svc>/tmp`) and restrict writes to that location.

**Added guidance (why `/run/<svc>/tmp` is often best):**
It is:

* scoped to the service
* cleared on reboot
* can be tightly permissioned
  This reduces risk from cross-service symlink attacks and accidental leakage across unrelated processes.



## Defining the Privilege Boundary in “Root Start → Drop Privileges”

### A. Keep Root-Only Work to the Minimum

Common examples that may require root briefly:

* Binding to ports below 1024 (though alternatives exist)
* Device access or privileged operations
* One-time initialization (ideally performed by the OS/service manager, not the daemon)

After completing privileged steps, drop privileges in the usual sequence:

* `setgid` → `initgroups` → `setuid`

After dropping privileges, the service should **not write to `/etc` or `/usr`**.

**Added implementation note (defense-in-depth):**
Dropping privileges is necessary but not sufficient. Pair it with filesystem policy so that even if an attacker gets code execution as `<svcuser>`, they cannot:

* modify configuration under `/etc`
* replace binaries/modules under `/usr`
* write to arbitrary parts of the filesystem



### B. Constrain Writes to a Small, Dedicated Set of Paths

A strong default policy is:

* Writable: **`/run/<svc>` and `/var/lib/<svc>`** (and optionally `/var/cache/<svc>`)
* Everything else: treat as **read-only**

This is a practical approach to enforce least privilege at the filesystem level.

**Added guidance (separate “data” from “inputs”):**
If the service accepts user-controlled uploads, consider separating them from core state:

* Core state: `/var/lib/<svc>/`
* Untrusted uploads: `/var/lib/<svc>/uploads/` (different ACLs/SELinux label/AppArmor profile if available)
  This helps prevent “data becomes code” problems (e.g., path traversal, template injection, unsafe deserialization, plugin loading from upload paths).



## systemd Implementation Notes (Operationally Strong and Low-Risk)

If you use systemd, declare directories and permissions in the **unit file** to reduce drift and eliminate manual setup errors.

Recommended unit directives:

* `User=<svcuser>` / `Group=<svcgroup>` (consider not running as root at all)
* `RuntimeDirectory=<svc>` → auto-creates `/run/<svc>`
* `StateDirectory=<svc>` → auto-creates `/var/lib/<svc>`
* `CacheDirectory=<svc>` → auto-creates `/var/cache/<svc>`
* `LogsDirectory=<svc>` → auto-creates `/var/log/<svc>`

Hardening examples:

* `ProtectSystem=strict`
* `ProtectHome=true`
* `ReadWritePaths=/var/lib/<svc> /run/<svc> ...`

This effectively turns writable locations into a **whitelist**, which is one of the strongest operational controls for services that drop privileges.

**Added note (why the systemd directives matter):**
Using `StateDirectory=` / `RuntimeDirectory=` etc. also enforces consistent ownership and mode at start, which reduces the risk of:

* manual directory creation mistakes
* permission drift across upgrades
* “it works on one host but not another” operational variance



## A Typical “Final Form” Example (myservice)

A clean, production-friendly layout might look like:

* `/etc/myservice/myservice.yaml` (configuration, managed by root)
* `/etc/myservice/credentials/` (secrets, root-managed with minimal access)
* `/usr/libexec/myservice/` (executables and private helper binaries)
* `/usr/lib/myservice/` (private libraries/modules, if applicable)
* `/run/myservice/` (pid, sockets, ephemeral runtime state)
* `/run/myservice/tmp/` (service-scoped temporary files)
* `/var/lib/myservice/` (persistent state)
* `/var/cache/myservice/` (rebuildable cache)
* `/var/log/myservice/` (only if file logs are required)



## Avoiding “Root Start” Entirely (Often the Best Option)

If the only reason for starting as root is “listen on 80/443,” consider safer alternatives:

* Grant `CAP_NET_BIND_SERVICE` to the binary
* systemd socket activation
* Terminate privileged ports in a reverse proxy (nginx/haproxy) and run the app on high ports

These approaches reduce attack surface substantially by eliminating long-lived root execution.

**Added note (principle of least privilege, applied):**
The goal is not only “drop privileges quickly,” but also “design the deployment so privileges are never needed.” When that is feasible, it typically yields the simplest, most auditable security posture.
