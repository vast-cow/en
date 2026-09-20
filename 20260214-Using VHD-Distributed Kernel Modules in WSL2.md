---
title: "Using VHD-Distributed Kernel Modules in WSL2"
description: "Practical Conclusions from Issue #12586   With the 6.6.y kernel series, WSL2 introduced a..."
pubDatetime: 2026-02-14T10:59:32.657Z
---

### Practical Conclusions from Issue #12586

With the 6.6.y kernel series, WSL2 introduced a structural change: **kernel modules are now distributed as a VHD/VHDX file** instead of being directly placed inside each distribution.

This shift affects how custom kernels, out-of-tree modules, and automated update systems should be implemented.

This article distills the practical conclusions from the following discussion:

> Reference:
> [https://github.com/microsoft/WSL/issues/12586](https://github.com/microsoft/WSL/issues/12586)

---

# Background: Why Modules Are Now in VHDX

Historically, kernel modules lived under:

```
/lib/modules/$(uname -r)
```

inside each distribution.

Starting with newer 6.6.y releases, Microsoft moved to shipping modules inside a **modules.vhdx** disk image. This decouples:

* `bzImage` (kernel binary)
* kernel modules
* distribution root filesystem

The goal appears to be cleaner packaging and easier global management of modules across distros.

However, early documentation did not clearly explain how to consume `modules.vhdx`, prompting the discussion in Issue #12586.

---

# Official Support: `kernelModules` in WSL 2.5.1+

As of **WSL 2.5.1.0 and later**, a new configuration key is supported:

```ini
[wsl2]
kernel=C:\\bzImage
kernelModules=C:\\modules.vhdx
```

### Requirements

* WSL **2.5.1.0 or newer**
* Earlier versions (2.4.x) interpret `kernelModules` as a boolean and produce an error

Example error on older versions:

```
Invalid boolean value 'C:\modules.vhdx' for key 'wsl2.kernelModules'
```

### Upgrade Notes

`wsl --update --pre-release` may not upgrade to 2.5.x automatically. In some cases, the 2.5.1 MSI must be installed manually.

After upgrading:

* The default kernel may move to Linux 6.6.x
* `kernelModules` becomes configurable
* A GUI option for custom module VHD may appear in WSL Settings

---

# Before 2.5.1: Manual Workarounds

Prior to official support, the process required manual steps such as:

1. `wsl --mount --vhd modules.vhdx`
2. Copying or bind-mounting into `/usr/lib/modules`
3. Creating systemd services to manage symlinks
4. Using Task Scheduler for auto-mounting

This approach was operationally heavy and fragile across multiple distributions.

WSL 2.5.1 eliminates that complexity.

---

# Current Limitation: Only One VHDX Is Supported

A central discussion point in the issue is whether multiple module images can be specified.

Currently, this is supported:

```ini
kernelModules=C:\\modules.vhdx
```

This is not supported:

```ini
kernelModules=C:\\modules.vhdx;C:\\modules-extra.vhdx
```

Only **a single VHDX file** can be mounted as the module source.

There is no:

* Overlay merging
* FUSE-based stacking
* Automatic union mount
* Multiple image list support

---

# Why Multiple VHDX Support Matters

Several real-world use cases require modular distribution:

* USB device-specific drivers
* Organization-specific internal modules
* Out-of-tree kernel modules
* Third-party hardware enablement packages

The desired model would allow:

* Stock kernel usage
* Independent module packs
* Layered module volumes

Without multi-VHD support, any additional modules must be merged into a single consolidated `modules.vhdx`.

---

# Important Clarification: Custom Modules Without Rebuilding the Kernel

There is a common misconception that custom modules require rebuilding the entire kernel.

This is not accurate.

The stock WSL kernel configuration is published, meaning:

* You can build out-of-tree modules against the stock kernel
* You can generate a `kernel-devel` style environment
* You do not need to maintain a custom kernel unless required

The constraint is packaging, not compatibility.

---

# Practical Deployment Strategy (Today)

Given the current limitations, the most robust solution is:

### Build and Consolidate

1. Build required modules against the stock kernel
2. Collect them into a single modules directory tree
3. Generate a unified `modules.vhdx`
4. Reference it via:

```ini
[wsl2]
kernelModules=C:\\modules.vhdx
```

This approach:

* Avoids kernel rebuilds
* Works with stock WSL kernel
* Keeps deployment predictable
* Simplifies distribution tooling (e.g., Scoop integration)

---

# What Would Improve the Design?

A potential enhancement would allow:

```ini
kernelModules=C:\\modules.vhdx;C:\\modules-extra.vhdx
```

with each volume mounted under:

```
/lib/modules/$(uname -r)/extra/vol1
/lib/modules/$(uname -r)/extra/vol2
```

and resolved via `depmod`.

However, this is not currently implemented.

---

# Summary

| Feature                  | Status                   |
| ------------------------ | ------------------------ |
| modules.vhdx support     | WSL 2.5.1+               |
| `.wslconfig` integration | Supported                |
| Multiple module VHDs     | Not supported            |
| Out-of-tree modules      | Supported                |
| Custom kernel required   | No                       |
| Recommended deployment   | Single consolidated VHDX |

---

# Final Takeaway

WSL 2.5.1 introduces proper support for VHD-based kernel modules via the `kernelModules` setting.

However:

* Only one module VHDX can be specified
* Multi-volume stacking is not supported
* Consolidation into a single module image remains the practical solution

For detailed context and ongoing discussion, see:

[https://github.com/microsoft/WSL/issues/12586](https://github.com/microsoft/WSL/issues/12586)
