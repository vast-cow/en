---
title: "How to avoid equation errors caused by `\\tag` when converting Markdown to Word using Pandoc."
description: "This article describes a method for resolving the issue of warnings appearing when converting Markdown to Word using Pandoc, which are caused by the LaTeX `\\tag{}` command. The solution involves using `sed` to remove the `\\tag{}` commands before the conversion process."
pubDatetime: 2026-11-19T18:16:00+09:00
---
Pandoc can be used to easily convert Markdown files to Word (DOCX) format. However, when converting Markdown files containing LaTeX equations, warnings may occur due to the `\tag{}` command used to specify equation numbers.

This time, I will introduce a method to ignore `\tag{}` using `sed` and convert Markdown to Word with Pandoc.

## The Problem

I tried to convert Markdown to Word with the following command:

```bash
pandoc -f markdown+tex_math_single_backslash doc.md -o doc.docx
```

However, the following warning was displayed:

```text
[WARNING] Could not convert TeX math ..., rendering as TeX:
  P_G\otimes P_L \tag{30}
                     ^
  unexpected control sequence \tag
```

The cause is `\tag{30}` included in the LaTeX equation within the Markdown.

`\tag{}` is a LaTeX command that explicitly specifies an equation number.

For example, it is used in equations such as the following:

```latex
$$
E = mc^2 \tag{1}
$$
```

When Pandoc converts such equations to Word's equation format (OMML), it may not be able to process `\tag` and a warning may occur.

## Solution: Remove `\tag` with `sed`

If the equation numbers are not needed, the problem can be solved by removing `\tag{}` before passing it to Pandoc.

Therefore, use the following command:

```bash
sed -E 's/\\tag\{[^}]*\}//g' doc.md | \
pandoc -f markdown+tex_math_single_backslash \
  -o doc.docx
```

This will remove `\tag{1}` or `\tag{30}` from the Markdown and execute the conversion to Word.

The original `doc.md` will not be changed.

## How the Command Works

### 1. Remove Equation Numbers with `sed`

```bash
sed -E 's/\\tag\{[^}]*\}//g' doc.md
```

This command removes the string matching `\tag{...}` in the Markdown file.

The meaning of the regular expression is as follows:

| Description | Meaning |
|---|---|
| `-E` | Use extended regular expressions |
| `s/...//g` | Delete all matching strings |
| `\\tag` | Matches the string `\tag` |
| `\{` | Matches `{` |
| `[^}]*` | Zero or more characters other than `}` |
| `\}` | Matches `}` |

For example, if the equation before conversion is as follows:

```latex
$$
E = mc^2 \tag{1}
$$
```

`sed` will transform it to:

```latex
$$
E = mc^2
$$
```

The equation body is retained, and only the number is removed.

### 2. Pass to Pandoc with a Pipe

```bash
| pandoc -f markdown+tex_math_single_backslash -o doc.docx
```

Using the pipe `|`, the text output by `sed` can be passed directly to Pandoc's standard input.

Therefore, it is not necessary to create a temporary Markdown file.

Also, `-f markdown+tex_math_single_backslash` is a specification to recognize LaTeX equations in `\(...\)` or `\[...\]` format in Markdown input.

Pandoc reads the modified Markdown and generates the Word file `doc.docx`.

## Advantages of this Method

- **Does not modify the original Markdown:** `doc.md` remains unchanged.
- **Can process in batch:** `\tag{}` in the file can be removed all at once.
- **No additional scripts are required:** It can be executed with `sed` and Pandoc only.
- **Can retain the equation body:** LaTeX code other than the equation number is generally passed through as is.

However, this method does not distinguish between whether it is inside an equation or not and removes all strings that match `\tag{...}` in the file. Also, it does not handle cases where there are nested curly braces within the tags.

## Summary

When converting Markdown to Word with Pandoc, warnings may occur during equation conversion due to LaTeX's `\tag{}`.

If the equation numbers are not needed, you can avoid this by removing them with `sed` before passing them to Pandoc.

The final command is as follows:

```bash
sed -E 's/\\tag\{[^}]*\}//g' doc.md | \
pandoc -f markdown+tex_math_single_backslash \
  -o doc.docx
```

This is a useful method when converting Markdown to Word from papers or technical documents containing many LaTeX equations.
