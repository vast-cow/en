---
title: "White Background Color Scheme for Windows Terminal"
description: "Overview   Windows Terminal allows you to customize its appearance through color schemes,..."
pubDatetime: 2026-04-20T09:22:01.109Z
---

## Overview

Windows Terminal allows you to customize its appearance through color schemes, improving both readability and overall usability. This article introduces a simple white background color scheme and explains its purpose and how to apply it.

This scheme uses a light background combined with muted text colors, making it suitable for extended use without excessive eye strain.

---

## Purpose of the Color Scheme

A white background color scheme offers several practical advantages:

* **Improved readability**: Dark text stands out clearly against a white background
* **Visual consistency**: Aligns well with common applications like editors and web browsers
* **Reduced eye fatigue**: Less harsh than dark themes in bright environments

It is particularly effective for daytime use or well-lit workspaces.

---

## Key Configuration Points

The provided configuration defines a balanced and minimal color palette:

* **Background (`background`)**: `#FFFFFF` (white)
* **Foreground (`foreground`)**: `#0C0C0C` (near-black)
* **Cursor (`cursorColor`)**: Dark color for clear visibility
* **Selection (`selectionBackground`)**: Dark highlight for contrast

Standard terminal colors (red, blue, green, etc.) and their bright variants are also included, ensuring proper color distinction in command outputs.

---

## How to Apply

### 1. Open the Settings File

Launch Windows Terminal and open the Settings. Edit the configuration file in JSON format.

### 2. Add the Color Scheme

Insert the following into the `"schemes"` array:

```json
"schemes": [
    {
        "name": "Theme Name",
        "background": "#FFFFFF",
        "foreground": "#0C0C0C"
        ...
    }
]
```

If a `"schemes"` section already exists, simply add this entry to the list.

---

### 3. Apply to a Profile

Assign the scheme to a profile (e.g., PowerShell or Command Prompt):

```json
"colorScheme": "Theme Name"
```

This activates the color scheme for that profile.

---

## Notes

* In dark environments, a white background may feel too bright
* Some tools rely on specific color assumptions, which may affect visibility

You can fine-tune individual colors as needed to better match your preferences.

---

## Summary

This white background color scheme provides a clean and readable interface for Windows Terminal. It is easy to implement and well-suited for everyday use, especially in bright environments. Adjust it as necessary to create a comfortable and efficient working setup.

```json
{
    "$help": "https://aka.ms/terminal-documentation",
    "$schema": "https://aka.ms/terminal-profiles-schema",
    "schemes": 
    [
        {
            "background": "#FFFFFF",
            "black": "#0C0C0C",
            "blue": "#0037DA",
            "brightBlack": "#767676",
            "brightBlue": "#3B78FF",
            "brightCyan": "#61D6D6",
            "brightGreen": "#16C60C",
            "brightPurple": "#B4009E",
            "brightRed": "#E74856",
            "brightWhite": "#F2F2F2",
            "brightYellow": "#C19C00",
            "cursorColor": "#0C0C0C",
            "cyan": "#3A96DD",
            "foreground": "#0C0C0C",
            "green": "#13A10E",
            "name": "My Light Theme",
            "purple": "#881798",
            "red": "#C50F1F",
            "selectionBackground": "#0C0C0C",
            "white": "#CCCCCC",
            "yellow": "#856B00"
        }
    ]
}
```
