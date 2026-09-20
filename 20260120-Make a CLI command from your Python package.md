---
title: "Make a CLI command from your Python package"
description: "In short: declare a console_scripts entry point in your package metadata, activate the target virtual..."
pubDatetime: 2026-01-20T02:16:56.274Z
updatedDate: 2026-01-21T09:29:48.461Z
---

In short: **declare a `console_scripts` entry point in your package metadata, activate the target virtual environment, then run `pip install` (during development use `pip install -e .`).** That will place an executable wrapper in `./venv/bin` (Unix/macOS) or `venv\Scripts` (Windows). Below are concrete examples and step-by-step instructions.



## 1) Example project layout

```
myproject/
├─ pyproject.toml   ← (or setup.cfg / setup.py)
└─ src/
   └─ cli.py
```

`src/cli.py` (example):

```py
def main():
    print("Hello from mycmd!")

if __name__ == "__main__":
    main()
```

The function referenced by the entry point (here `mypackage.cli:main`) will be called when the installed command runs.



## 2) Recommended: `pyproject.toml` + setuptools

When using setuptools as the build backend, add a `pyproject.toml` like:

```toml
[build-system]
requires = ["setuptools>=61", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "mypackage"
version = "0.0.1"
description = "example package"

[project.scripts]
mycmd = "cli:main"
```

This makes `mycmd` an installed command that calls `cli:main`.



## 3) Another common way: `setup.cfg`

If your project uses `setup.cfg`:

```ini
[metadata]
name = mypackage
version = 0.0.1

[options]
packages = find:

[options.entry_points]
console_scripts =
    mycmd = cli:main
```

(Older projects may still use `setup.py` with `entry_points={...}`.)



## 4) Install into a virtual environment (activate first!)

Always create and activate a venv before installing, so the wrapper lands in the venv and not in system Python.

Unix / macOS:

```bash
python -m venv venv
source venv/bin/activate
pip install -e .
```

Windows (PowerShell):

```powershell
python -m venv venv
venv\Scripts\Activate.ps1
pip install -e .
```

`-e` (editable) is convenient during development because code edits are immediately reflected when you run the command. Using `pip install .` (non-editable) also creates the script/binary but installs a fixed build.



## 5) Verify the installed command

Unix/macOS:

```bash
which mycmd         # shows venv/bin/mycmd if venv is active
mycmd               # run it and check output
```

Expected locations:

* Unix/macOS: `./venv/bin/mycmd` (executable wrapper)
* Windows: `venv\Scripts\mycmd.exe` (or `mycmd-script.py`)

Run `mycmd` and you should see `Hello from mycmd!` (or whatever your `main()` prints).



## 6) Notes & best practices

* **Always activate the virtual environment before `pip install`** unless you explicitly want a system/global install.
* `console_scripts` (entry points) are the recommended approach — they produce OS-appropriate wrapper scripts and integrate with package metadata.
* `scripts=` (legacy `setup.py` option) copies scripts directly; entry points are generally better for dependency resolution and cross-platform behavior.
* Libraries like Click or Typer make building richer CLI apps simple; the entry point syntax remains the same (point to the function that invokes the CLI).
