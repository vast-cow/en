---
title: "How to Standardize Line Endings in Git Without Committing `.gitattributes`"
description: "When developing with Git, differences between CRLF and LF can become an issue when working across..."
pubDatetime: 2026-08-17T06:02:23.416Z
---

When developing with Git, **differences between CRLF and LF** can become an issue when working across Windows and Linux/macOS.

A common solution is to place a `.gitattributes` file in the repository to define line-ending rules.

However, there are also cases where you may want to:

* Avoid adding `.gitattributes` to the repository
* Standardize line endings only in your own environment
* Apply the same rules across multiple repositories

Git allows you to configure equivalent settings without committing `.gitattributes` to the repository.

## Configure Only a Specific Repository

If you want the settings to apply only to a specific repository, you can use the following file:

```text
.git/info/attributes
```

Because this file is located inside the `.git` directory, it is normally not tracked by Git.

For example, you can configure it as follows:

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.cmd text eol=crlf
```

With this configuration, text files are generally treated as LF, while Windows batch files use CRLF.

Here is what each setting means:

```gitattributes
* text=auto eol=lf
```

For files recognized as text, this sets the line endings in the working tree to LF.

Meanwhile,

```gitattributes
*.bat text eol=crlf
*.cmd text eol=crlf
```

treats `.bat` and `.cmd` files as text files and uses CRLF for them in the working tree.

In other words, the basic policy is:

```text
Regular text files → LF
Windows batch files → CRLF
```

## Configure Common Settings for All Repositories

If creating `.git/info/attributes` every time is inconvenient, you can use Git's global configuration.

First, configure the location of the attributes file:

```bash
git config --global core.attributesFile ~/.gitattributes
```

Then create `~/.gitattributes`:

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.cmd text eol=crlf
```

This allows you to apply a common attributes configuration to Git repositories used in your environment.

## Choosing Between `.git/info/attributes` and Global Configuration

It is useful to choose the appropriate method based on your needs.

| Method                 | Scope                       | Git-tracked                            |
| ---------------------- | --------------------------- | -------------------------------------- |
| `.gitattributes`       | Repository                  | Yes                                    |
| `.git/info/attributes` | Specific repository         | No                                     |
| `core.attributesFile`  | Your entire Git environment | Not tracked by individual repositories |

If the entire team needs to use the same rules, placing `.gitattributes` in the repository is the appropriate approach.

On the other hand, for personal settings such as "I want LF to be the default only in my development environment," it is convenient to configure:

```bash
git config --global core.attributesFile ~/.gitattributes
```

If you need exceptional rules for only a specific repository, you can use:

```text
.git/info/attributes
```

## Summary

Even if you cannot add `.gitattributes` to the repository, you can still use Git's attributes mechanism.

For settings shared across all repositories, configure:

```bash
git config --global core.attributesFile ~/.gitattributes
```

and set:

```gitattributes
* text=auto eol=lf
*.bat text eol=crlf
*.cmd text eol=crlf
```

For only a specific repository, put the same content in:

```text
.git/info/attributes
```

This allows you to apply the rule **LF by default, with CRLF only for Windows batch files** without committing a configuration file to the repository.
