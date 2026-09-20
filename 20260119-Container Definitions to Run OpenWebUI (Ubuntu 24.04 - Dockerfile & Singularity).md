---
title: "Container Definitions to Run OpenWebUI (Ubuntu 24.04 / Dockerfile & Singularity)"
description: "Unified base OS: Both Docker and Singularity use Ubuntu 24.04 to keep the runtime environment..."
pubDatetime: 2026-01-19T11:27:09.921Z
---

* **Unified base OS:** Both Docker and Singularity use **Ubuntu 24.04** to keep the runtime environment consistent.
* **Minimal dependency set:** Installs `python3` and `python3-venv`, and `ffmpeg` for audio/video handling, using **no-install-recommends**.
* **Reproducible, non-interactive builds:** Fixes timezone to **UTC** and avoids interactive prompts via `DEBIAN_FRONTEND=noninteractive` (ARG in Dockerfile; applied during install in Singularity). Also cleans apt caches (`apt-get clean` and removing `/var/lib/apt/lists/*`) to reduce image size.

```apptainer
Bootstrap: docker
From: ubuntu:24.04

%post
    set -eux

    # timezone: UTC
    ln -sf /usr/share/zoneinfo/UTC /etc/localtime

    # packages
    apt-get update
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends python3 python3-venv ffmpeg

    apt-get clean
    rm -rf /var/lib/apt/lists/*
```

```dockerfile
# syntax=docker/dockerfile:1
FROM ubuntu:24.04

RUN set -eux; \
    # timezone: UTC
    ln -sf /usr/share/zoneinfo/UTC /etc/localtime; \
    \
    # packages
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends python3 python3-venv ffmpeg; \
    apt-get clean; \
    rm -rf /var/lib/apt/lists/*
```
