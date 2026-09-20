---
title: "Make PowerShell Readable on a White Windows Terminal Background"
description: "When you switch Windows Terminal to a light or white background, the default PowerShell token colors..."
pubDatetime: 2026-04-20T09:16:08.952Z
---

When you switch Windows Terminal to a light or white background, the default PowerShell token colors can become hard to read. You can fix this by customizing **PSReadLine**’s color scheme so that prompts, selections, and syntax tokens use higher-contrast tones.

Below is a minimal configuration that keeps most text at the terminal’s default foreground color, uses a dim gray for numbers, and highlights selections in an inverted style. It also gives commands a warm accent for visibility.

```powershell
# Run in PowerShell (requires PSReadLine)
Import-Module PSReadLine
Set-PSReadLineOption -Colors @{
    ContinuationPrompt = "$([char]0x1b)[39m"      # default foreground
    Default            = "$([char]0x1b)[39m"      # plain text
    Type               = "$([char]0x1b)[39m"      # type names
    Selection          = "$([char]0x1b)[39;49;7m" # inverse (swap fg/bg)
    Command            = "$([char]0x1b)[33m"      # yellow for commands
    Member             = "$([char]0x1b)[39m"      # .member names
    Number             = "$([char]0x1b)[38;5;242m"# ANSI 256-color gray
}
```

## Why these choices?

* **[39m] default foreground**: Lets Windows Terminal’s color scheme control the actual text color, so it stays readable whether the background is light or dark.
* **[33m] yellow for commands**: A standard, high-contrast accent that stands out on white without being harsh.
* **[38;5;242m] gray for numbers**: Uses the 256-color palette to render numbers in a softer gray that remains legible on light backgrounds.
* **[7m] inverse for selections**: Inverts foreground/background so selections “pop” regardless of theme.

## How to apply permanently

1. Open your PowerShell profile:

   ```powershell
   notepad $PROFILE
   ```
2. Paste the `Set-PSReadLineOption -Colors` block into the profile file and save.
3. Restart the terminal (or run the block once) to apply.

That’s it—now your PowerShell prompt and syntax colors stay crisp and readable even with a white Windows Terminal theme.
