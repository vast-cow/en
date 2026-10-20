---
title: "How to manually adjust the indentation of a selected region in Emacs: Using indent-rigidly"
description: "Describes how to use Emacs' indent-rigidly command to shift selected lines left or right while preserving relative indentation."
pubDatetime: 2026-10-20T12:41:00+09:00
---

When editing code or text in Emacs, you may want to shift multiple lines of indentation to the right or left at once.

The Emacs standard command **`indent-rigidly`** is useful for this.

This article introduces how to manually adjust the indentation of a selected range and how to finish the operation after adjustment.

## 1. What is `indent-rigidly`?

`indent-rigidly` is a command that changes the indentation of a selected region at once.

While the regular `indent-region` automatically adjusts the indentation according to the syntax of the programming language, `indent-rigidly` allows you to manually move the indentation left or right while preserving the relative indentation of each line.

For example, consider the following code:

```python
if condition:
    print("Hello")
    print("World")
```

If you shift these three lines to the right by 4 characters, it will look like this:

```python
    if condition:
        print("Hello")
        print("World")
```

The key feature is that the relative indentation of each line is maintained.

## 2. Manually Adjusting the Indentation of a Selected Range

The steps are as follows:

1. Start range selection with `C-SPC`.
2. Move the cursor to select the target lines.
3. Execute `indent-rigidly` with `C-x TAB`.
4. Adjust the indentation with the left and right arrow keys.

After executing `indent-rigidly`, you can adjust it with the following keys:

| Key | Action |
|---|---|
| `→` | Move 1 character to the right |
| `←` | Move 1 character to the left |
| `S-→` | Move `tab-width` characters to the right |
| `S-←` | Move `tab-width` characters to the left |

`S-→` and `S-←` are performed by pressing the Shift key while pressing the left and right arrow keys.

*Note: The interactive adjustment with the arrow keys may behave differently depending on the Emacs version and settings.*

## 3. How to End Indentation Adjustment?

After executing `C-x TAB` and interactively adjusting the indentation, press **`RET` (Enter key)** to confirm the adjustment and exit.

The basic flow is as follows:

```text
C-SPC      Start range selection
Cursor movement  Select the range
C-x TAB    Execute indent-rigidly
→ → →      Move 3 characters to the right
RET        Confirm the adjustment and exit
```

If you want to interrupt the operation, use `C-g`.

However, pressing `C-g` does not necessarily automatically revert the indentation changes made up to that point.

If you want to undo the changes, use `undo` (usually `C-/`) as needed.

## 4. Changing Indentation by a Specified Number of Characters

In addition to interactive adjustment, you can also specify a number to change the indentation at once.

**To shift 4 characters to the right:**

```text
C-u 4 C-x TAB
```

**To shift 4 characters to the left:**

```text
C-u - 4 C-x TAB
```

This is useful when you know the number of characters you want to change.

## 5. Difference from `indent-region`

Emacs has another command for adjusting indentation called `indent-region`.

| Command | Key Binding | Purpose |
|---|---|---|
| `indent-rigidly` | `C-x TAB` | Manually shift the selected range left or right |
| `indent-region` | `C-M-\` | Automatically format according to syntax rules |

For example, if you want to format code such as Python or C according to the syntax, `indent-region` is suitable.

On the other hand, if you want to move multiple lines to the right or left without changing the code structure, `indent-rigidly` is useful.

## Summary

To manually adjust the indentation of a selected range in Emacs, use `indent-rigidly`.

The operations to remember are as follows:

| Operation | Key |
|---|---|
| Start range selection | `C-SPC` |
| Manual indentation adjustment | `C-x TAB` |
| Move right | `→` |
| Move left | `←` |
| Confirm adjustment and exit | `RET` |
| Interrupt operation | `C-g` |
| Move 4 characters to the right | `C-u 4 C-x TAB` |
| Move 4 characters to the left | `C-u - 4 C-x TAB` |

The most important thing to remember is to **start adjusting with `C-x TAB` and end with `RET`**.

It is useful when you want to make fine manual adjustments to the indentation of multiple lines.
