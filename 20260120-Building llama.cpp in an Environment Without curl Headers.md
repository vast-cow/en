---
title: "Building llama.cpp in an Environment Without curl Headers"
description: "When you try to build llama.cpp on a system where the curl development headers are not installed, the..."
pubDatetime: 2026-01-20T02:57:25.423Z
---

When you try to build **llama.cpp** on a system where the **curl development headers** are not installed, the build may fail because the compiler cannot find curl’s header files (such as `curl/curl.h`). One straightforward workaround is to download the matching curl source package (so you have the headers locally) and then point CMake to the existing curl **library** on your system plus the downloaded **include** directory.

Below is a simple step-by-step example using curl **7.76.1**.

## 1) Check the Installed curl Version

First, confirm which curl version is available in your environment:

```bash
curl --version
```

Example output:

```text
curl 7.76.1 ...
```

## 2) Confirm Which libcurl curl Is Actually Using

Before configuring the build, it is useful to confirm the exact `libcurl.so` path used by the `curl` binary. This helps you set `CURL_LIBRARY` correctly.

Run:

```bash
ldd $(which curl) | grep curl
        libcurl.so.4 => /lib64/libcurl.so.4 (0x00007faa934a6000)
```

In this example, the relevant library path is:

* `/lib64/libcurl.so.4`

## 3) Download the Matching curl Source Archive

Download the curl source tarball for version 7.76.1 from GitHub:

```bash
curl -LO https://github.com/curl/curl/archive/refs/tags/curl-7_76_1.tar.gz
```

## 4) Extract the Archive

Unpack the downloaded file:

```bash
tar xvf curl-7_76_1.tar.gz
```

This will create a directory such as `curl-curl-7_76_1/` containing the source code and, importantly, the `include/` directory.

## 5) Configure the Build with CMake

Now configure your build so that CMake uses:

* The system-installed curl library (example: `/lib64/libcurl.so.4`)
* The headers from the extracted curl source directory

Run:

```bash
cmake -B build -DCURL_LIBRARY=/lib64/libcurl.so.4 -DCURL_INCLUDE_DIR=./curl-curl-7_76_1/include
```

Notes:

* `-B build` tells CMake to generate build files into the `build/` directory.
* `-DCURL_LIBRARY=...` points to the libcurl shared library already present on the machine.
* `-DCURL_INCLUDE_DIR=./curl*/include` uses a wildcard to match the extracted curl source folder and selects its `include/` directory.

## 6) Build

Finally, compile using multiple jobs:

```bash
cmake --build build -j
```

At this point, the build should be able to find curl headers from the extracted source package while linking against the system’s libcurl library, avoiding the need to install curl development packages in the environment.
