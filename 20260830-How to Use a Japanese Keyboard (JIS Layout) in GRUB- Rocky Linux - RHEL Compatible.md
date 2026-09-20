---
title: "How to Use a Japanese Keyboard (JIS Layout) in GRUB: Rocky Linux / RHEL Compatible"
description: "Generate and test a Japanese GRUB keymap, make it persistent through custom boot configuration, and distinguish it from Linux console settings."
pubDatetime: 2026-08-30T07:46:21.679Z
---

# How to Use a Japanese Keyboard (JIS Layout) in GRUB: Rocky Linux / RHEL Compatible

When working with the GRUB menu or command line, you may encounter issues where the keyboard is treated as a US layout.

This is particularly problematic with Japanese keyboards, as:

*   `@`
*   `:`
*   `"`
*   `\`
*   `_`

These symbols are located in different positions on US and JIS layouts, making it quite inconvenient to edit kernel parameters or enter commands in GRUB.

This article summarizes how to change the GRUB keyboard layout to the Japanese JIS layout on RHEL-based Linux distributions such as Rocky Linux, AlmaLinux, RHEL, and CentOS Stream.

Note that the "Japanese keyboard" referred to here does not refer to the Japanese input IME. It is not intended to enable Japanese character input in GRUB, but rather to configure it to match the physical key layout of a JIS keyboard.

---

## Linux Keyboard Settings and GRUB Keyboard Settings Are Different

Normally, you can change the Linux console keymap with the following command:

```bash
localectl set-keymap jp
```

Alternatively, you can specify it in the kernel parameters or configuration file, such as:

```text
vconsole.keymap=jp
```

However, these are settings that take effect after the Linux kernel has booted.

Since GRUB runs before Linux, the Linux keyboard settings are not reflected in GRUB.

In other words, in the boot sequence:

```text
GRUB
  ↓
Linux kernel
  ↓
systemd / console
```

The keyboard layout for GRUB must be configured within GRUB itself.

---

# Packages Required for Rocky Linux / RHEL

In Rocky Linux and RHEL-based systems, GRUB-related commands have `grub2-` prefixes.

Install the necessary packages:

```bash
sudo dnf install grub2-tools-extra kbd
```

The following commands are mainly used:

```text
grub2-kbdcomp
grub2-mkconfig
```

Use `grub2-kbdcomp` to create a keyboard layout file that GRUB can read.

---

# Create a Japanese Keymap for GRUB

First, create a save location:

```bash
sudo mkdir -p /boot/grub2/layouts
```

Next, create the Japanese JIS layout keymap:

```bash
sudo grub2-kbdcomp -o /boot/grub2/layouts/jp.gkb jp
```

Verify that it was created:

```bash
ls -l /boot/grub2/layouts/jp.gkb
```

If a file exists as follows, the preparation is complete:

```text
/boot/grub2/layouts/jp.gkb
```

In GRUB, the keyboard layout is changed by loading this `.gkb` file.

---

# Temporarily Change to JIS Layout Only During GRUB Boot

Instead of making a permanent change all at once, it is recommended to first load it temporarily from the GRUB command line and verify that it works.

When the GRUB menu is displayed, press:

```text
c
```

to enter the GRUB command line.

Execute the following command:

```grub
insmod keylayouts
keymap jp
```

This will change to the Japanese JIS layout for that GRUB session only.

Since it will revert to the original upon reboot, it is suitable for testing the settings.

---

## If `keymap jp` Cannot Be Loaded

Check the `prefix` used by GRUB:

```grub
echo $prefix
```

For example, it may display:

```text
(hd0,gpt2)/grub2
```

Next, check if the keymap is visible:

```grub
ls $prefix/layouts/
```

If:

```text
jp.gkb
```

is displayed, GRUB can access the file.

In this case, it should normally be loaded with:

```grub
keymap jp
```

Depending on the environment, you may want to explicitly specify the file to confirm:

```grub
keymap $prefix/layouts/jp.gkb
```

---

# Make the JIS Layout Permanent

Once you have verified that it works, incorporate it into the GRUB configuration.

On RHEL-based systems, appending to `/etc/grub.d/40_custom` is a clear method.

```bash
sudo vi /etc/grub.d/40_custom
```

Add the following content:

```grub
insmod keylayouts
keymap ${prefix}/layouts/jp.gkb
```

Alternatively, depending on the environment, the following specification may also work:

```grub
insmod keylayouts
keymap jp
```

Personally, I prefer to explicitly specify it so that it is clear which file is being used:

```grub
keymap ${prefix}/layouts/jp.gkb
```

This makes it easier to troubleshoot.

---

# Regenerate grub.cfg

After changing the settings, regenerate the GRUB configuration.

On newer RHEL-based systems such as Rocky Linux 9:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

Reboot:

```bash
sudo reboot
```

In the GRUB menu, press `c` to open the command line and check the symbol keys.

For example:

```text
@
:
"
\
_
```

If these can be entered as they are printed on the keycaps, the JIS layout has been applied.

---

# Notes for UEFI Environments

Older articles mention examples of running `grub2-mkconfig` directly on the following file in UEFI environments:

```text
/boot/efi/EFI/redhat/grub.cfg
```

In Rocky Linux, it may be:

```text
/boot/efi/EFI/rocky/grub.cfg
```

However, in recent RHEL-based systems, the UEFI `grub.cfg` is configured to be used as a stub to load the main `/boot/grub2/grub.cfg`.

Therefore, in Rocky Linux 9 and later, it is generally recommended to use:

```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

It is safer not to directly overwrite the UEFI `grub.cfg` unnecessarily.

---

# Is `terminal_input at_keyboard` Necessary?

In GRUB keyboard configuration examples, you may see settings like:

```grub
insmod at_keyboard
terminal_input at_keyboard
```

However, on USB keyboards and UEFI environments, explicitly switching the input path to `at_keyboard` may prevent keyboard input.

If you only want to change to the JIS layout, it is safest to first try:

```grub
insmod keylayouts
keymap ${prefix}/layouts/jp.gkb
```

Unless there is a specific reason, it is not necessary to change the input terminal itself.

---

# Temporary Change vs. Permanent Change

To summarize, the following applies:

| Method                 | Setting                                | After Reboot |
| ---------------------- | ------------------------------------- | ------------ |
| GRUB command line      | `insmod keylayouts` → `keymap jp` | Reverts      |
| Add to `40_custom`     | `keymap ${prefix}/layouts/jp.gkb` | Persists     |
| Linux `localectl`      | Changes only the Linux console        | Does not affect GRUB |

It is safe to test with a temporary change first, and then make it permanent if there are no problems.

---

# Can I Change to the JIS Layout in GRUB Without Creating a `.gkb` File?

Basically, no.

The `keymap` command in GRUB is designed to load pre-created GRUB keyboard layout files.

Therefore, during Linux startup, you need to:

```bash
grub2-kbdcomp
```

to create:

```text
jp.gkb
```

in advance.

At a minimum, you need to perform the following preparations on the Linux side:

```bash
sudo mkdir -p /boot/grub2/layouts
sudo grub2-kbdcomp -o /boot/grub2/layouts/jp.gkb jp
```

After that, you can always run:

```grub
insmod keylayouts
keymap jp
```

in the GRUB command line to temporarily switch to the JIS layout.

---

# Summary

When setting GRUB to a Japanese JIS keyboard layout on Rocky Linux or RHEL, the key points are:

1.  First, create a GRUB keymap:

    ```bash
    sudo dnf install grub2-tools-extra kbd
    sudo mkdir -p /boot/grub2/layouts
    sudo grub2-kbdcomp -o /boot/grub2/layouts/jp.gkb jp
    ```

2.  If you want to try it temporarily in GRUB:

    ```grub
    insmod keylayouts
    keymap jp
    ```

3.  To make it permanent, add to `/etc/grub.d/40_custom`:

    ```grub
    insmod keylayouts
    keymap ${prefix}/layouts/jp.gkb
    ```

    and run:

    ```bash
    sudo grub2-mkconfig -o /boot/grub2/grub.cfg
    ```

GRUB runs before Linux boots, so the `localectl set-keymap jp` and other Linux settings are completely separate.

If you frequently edit kernel parameters or perform rescue operations in GRUB, setting up the JIS keyboard will make it much easier to operate.
