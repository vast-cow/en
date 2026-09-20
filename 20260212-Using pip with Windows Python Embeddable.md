---
title: "Using pip with Windows Python Embeddable"
description: "The Windows Python embeddable distribution (python-3.x.x-embed-amd64.zip) is a lightweight,..."
pubDatetime: 2026-02-12T11:52:17.837Z
---

The **Windows Python embeddable distribution** (`python-3.x.x-embed-amd64.zip`) is a lightweight, self-contained runtime designed primarily for application embedding and redistribution.

However, it does **not include pip or venv by default**.

This article explains:

1. How to install **pip** using `get-pip.py` (Method A)
2. How to configure `_pth` correctly (critical step)
3. How to use **virtualenv as a replacement for venv**

---

## Target Environment

* Windows
* `python-3.x.x-embed-amd64.zip`
* Example: Python 3.11.x

---

# Step 1 — Extract the Embeddable Distribution

Download the embeddable ZIP from python.org and extract it to a directory.

Example:

```
C:\python311-embed\
```

Confirm that `python.exe` exists in that directory.

---

# Step 2 — Download get-pip.py

Open PowerShell inside the extracted directory and run:

```powershell
curl -o get-pip.py https://bootstrap.pypa.io/get-pip.py
```

---

# Step 3 — Install pip

Run:

```powershell
.\python.exe .\get-pip.py
```

Verify installation:

```powershell
.\python.exe -m pip --version
```

If a version number is displayed, pip is installed successfully.

---

# Step 4 — Modify the `_pth` File (Critical)

The embeddable distribution restricts the module search path via a `_pth` file.

Without modification, packages installed via pip will not be importable.

## Locate the `_pth` file

Example:

```
python311._pth
```

## Required changes

1. Uncomment `import site`
2. Add `Lib\site-packages`

### Example configuration:

```
python311.zip
.
Lib\site-packages
import site
```

This enables Python to discover installed packages.

---

# Step 5 — Test Package Installation

Install a package:

```powershell
.\python.exe -m pip install requests
```

Test it:

```powershell
.\python.exe
>>> import requests
>>> requests.__version__
```

If no errors occur, the setup is complete.

---

# venv Is Not Available — Use virtualenv Instead

The embeddable distribution typically does **not include**:

* `venv`
* `ensurepip`

Therefore, the standard `python -m venv` command does not work.

## Solution: Install virtualenv

After pip is installed:

```powershell
.\python.exe -m pip install virtualenv
```

## Create a virtual environment

```powershell
.\python.exe -m virtualenv venv
```

## Activate it

```powershell
.\venv\Scripts\activate
```

You now have an isolated Python environment similar to what `venv` provides in a standard installation.

---

# venv vs virtualenv (Quick Comparison)

| Feature                     | venv                  | virtualenv |
| --------------------------- | --------------------- | ---------- |
| Included in standard Python | Yes                   | No         |
| Available in embeddable     | No                    | Yes        |
| Requires pip                | No (standard install) | Yes        |
| Recommended for embeddable  | No                    | Yes        |

For the embeddable distribution, **virtualenv is the practical replacement for venv**.

---

# Common Issues

### pip install succeeds but import fails

Cause:

* `Lib\site-packages` missing in `_pth`
* `import site` still commented out

### SSL certificate errors

The embeddable distribution may lack full certificate configuration.
Additional configuration (e.g., `certifi`) may be required in some environments.

---

# Summary

To use pip with the Windows Python embeddable distribution:

1. Install pip via `get-pip.py`
2. Modify the `_pth` file to enable `site-packages`
3. Use `virtualenv` instead of `venv` for isolated environments

Although the embeddable distribution is designed primarily for redistribution scenarios, it can be adapted for development use with proper configuration.
