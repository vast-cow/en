---
title: "Workaround for Certbot Installation on Raspbian 11 (Bullseye) armv7l with Python 3.9"
description: "Pin Dependencies to Avoid pip Install Failures   On Raspbian 11 (Bullseye) running on..."
pubDatetime: 2026-01-19T09:47:23.419Z
---

## Pin Dependencies to Avoid pip Install Failures

On Raspbian 11 (Bullseye) running on armv7l, installing Certbot via `pip` under Python 3.9 can fail due to build and dependency compatibility issues. A practical workaround is to pin versions explicitly so `pip` resolves a known-good combination. In particular, locking `cffi` to a compatible release helps prevent compilation or wheel resolution errors that block Certbot installation.

Use the following approach to proceed reliably:

```bash
pip install "cffi<2.0.0" certbot
```

* Pin `cffi` to a specific version known to work on armv7l
* Install a matching Certbot version in the same command
* Prefer a clean virtual environment to avoid conflicting packages
* Re-run with `--no-cache-dir` if you suspect cached artifacts
* Verify the installed versions after completion

### Alternative: Install libffi-dev to Build Newer cffi

As an alternative approach, you can install the system development package for `libffi` so that `cffi==2.0.0` can be built/installed successfully even on armv7l environments.

```bash
sudo apt update
sudo apt install -y libffi-dev
pip install "cffi==2.0.0"
```

If you still encounter build-related issues, also ensure basic build tooling is present (e.g., `build-essential` and Python headers), and retry with `--no-cache-dir`.

This version-pinning strategy provides a stable path to install Certbot on constrained ARM environments without changing the OS or Python version.
