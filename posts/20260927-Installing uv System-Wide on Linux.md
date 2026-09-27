---
pubDatetime: 2026-09-27T21:58:00+09:00
title: "Installing uv System-Wide on Linux"
description: "Specify the installation directory for the official installer as `/usr/local/bin` to allow all users to run `uv` / `uvx`. Also, prevent the installer from modifying the user-specific PATH."
---

If you want to install `uv` **system-wide** on Linux and make `uv` / `uvx` available to all users, the simplest method is to install it in `/usr/local/bin`.

```bash
curl -LsSf https://astral.sh/uv/install.sh | UV_INSTALL_DIR=/usr/local/bin UV_NO_MODIFY_PATH=1 sudo -E sh
```

Verification:

```bash
which uv
uv --version
which uvx
```

Expected output:

```text
/usr/local/bin/uv
/usr/local/bin/uvx
```

The official installer normally installs to the user's home directory, but you can change the installation location with `UV_INSTALL_DIR`. Setting `UV_NO_MODIFY_PATH=1` also prevents the installer from modifying the PATH in files like root's `.bashrc`.

### Important Notes

This configuration makes the **`uv` command itself available system-wide**. It does not automatically make Python, caches, or tools installed with `uv tool install` available in a shared directory. By default, these will still be stored in user-specific directories like `~/.local/share/uv` and `~/.cache/uv`.

For example:

```bash
# user1
uv python install 3.13

# user2
uv python list
```

In this case, `user1` and `user2` will have separate management areas.

If your intention is to **make not only the `uv` command itself, but also the Python and tools installed by `uv` available in a shared directory, such as `/opt/uv`**, for all users, it is better to create a system-wide configuration for Rocky Linux, including `/etc/profile.d/uv.sh`. In this case, you can design the directory structure to separate them, for example, `/opt/uv/{python,tools,cache}`.
