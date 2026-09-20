---
title: "When did WSL's --import add support for xz and zstd?"
description: "The conclusion is: I could not find an official release note that says \"supported from this version..."
pubDatetime: 2026-06-18T07:50:42.832Z
updatedDate: 2026-06-18T08:13:49.579Z
---

The conclusion is: **I could not find an official release note that says "supported from this version onward."** However, there are lower bounds that can be verified from publicly available information.

| Format     | Earliest verifiable evidence from public sources | Confidence      |
| ---------- | -----------------------------------------------: | --------------- |
| `.tar.xz`  |                **WSL 1.1.3.0, as of 2023-03-12** | Relatively high |
| `.tar.zst` |                  **WSL 2.1.5, as of 2024-04-11** | Moderate        |

## `.tar.xz` Was Working at Least in WSL 1.1.3.0

In GitHub issue #9778, the reported environment is **WSL Version 1.1.3.0**, and the reproduction steps include:

`wsl --import ... lunar-server-cloudimg-amd64-root.tar.xz`

The issue itself is not about import failures but about problems opening the imported distribution from VS Code afterward. That suggests that using `.tar.xz` with `wsl --import` was already a valid workflow at that time. ([GitHub][1])

## `.tar.zst` Was Working at Least in WSL 2.1.5

In a comment on GitHub issue #6056, dated **2024-04-11**, a user reported that on **WSL 2.1.5 (latest at the time)**, importing `.tar.gz`, `.tar.xz`, and `.tar.zst` all worked.

This is a community report rather than an official release note, but it is still useful as the earliest confirmation I could find for `.tar.zst` support. ([GitHub][2])

## As of 2020, Support Was Not Yet an Explicitly Supported Feature

Issue #6056, opened in October 2020, was a feature request asking for `.xz` support in `wsl.exe --import` and the WSL Distro Launcher ecosystem.

The issue notes that the expected formats at the time were `install.tar` and gzip-compressed archives such as `install.tar.gz` or `install.tgz`, and requests support for XZ-compressed archives.

In other words, **as of 2020, `.tar.xz` was at least not an explicitly documented or officially assumed format**. ([GitHub][3])

## Internally, Import Does Not Appear to Hard-Code Compression Formats

Looking at the current WSL source code, the `wsl --import` path appears to reject `.vhd` and `.vhdx` when passed without `--vhd`, then opens the file and passes it to `RegisterDistribution`.

The `wsl.exe` client code does not appear to explicitly branch on `.gz`, `.xz`, or `.zst` extensions. ([GitHub][4])

By contrast, the `--export` side exposes explicit format options such as:

* `tar.gz`
* `tar.xz`
* `tar`
* `vhd`

This suggests that **export explicitly enumerates output formats, while import largely hands a tar-based archive to backend extraction logic and lets that layer handle the compression format**. ([GitHub][4])

## Summary

Strictly speaking:

> `.tar.xz` was being used with `wsl --import` no later than **WSL 1.1.3.0 (March 2023)**.
>
> `.tar.zst` has confirmed working reports no later than **WSL 2.1.5 (April 2024)**.
>
> However, no official changelog has been found that explicitly states when support for either format was first introduced.

Therefore, **support did not first appear in WSL 2.5.x**. In particular, `.tar.xz` appears to have been working significantly earlier.

[1]: https://github.com/microsoft/WSL/issues/9778 "Problem running code and code-insiders on Ubuntu distro 23.04 · Issue #9778 · microsoft/WSL · GitHub"
[2]: https://github.com/microsoft/WSL/issues/6056?utm_source=chatgpt.com "Add .xz archive support to wsl.exe --import to improve ..."
[3]: https://github.com/microsoft/WSL/issues/6056?timeline_page=1 "Add .xz archive support to wsl.exe --import to improve microsoft/WSL-DistroLauncher · Issue #6056 · microsoft/WSL · GitHub"
[4]: https://github.com/microsoft/WSL/blob/master/src/windows/common/WslClient.cpp "WSL/src/windows/common/WslClient.cpp at master · microsoft/WSL · GitHub"
