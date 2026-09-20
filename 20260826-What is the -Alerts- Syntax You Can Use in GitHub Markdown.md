---
title: "What is the \"Alerts\" Syntax You Can Use in GitHub Markdown?"
description: "When reading GitHub READMEs or Issues, you may come across syntax like this:    &gt;..."
pubDatetime: 2026-08-26T06:55:58.209Z
---

When reading GitHub READMEs or Issues, you may come across syntax like this:

```md
> [!IMPORTANT]
> **If you are upgrading from a previous version**
>
> The configuration method has changed.
> Please check the migration procedure before upgrading.
```

This is not a regular Markdown quote; it's a Markdown extension called **Alerts** that GitHub supports.

On GitHub, it is displayed as a note box with icons and decorations according to the type, such as `IMPORTANT`.

## It's Not Standard Markdown

First, it's important to understand this part:

```md
>
```

This is the standard Markdown quote syntax.

On the other hand,

```md
[!IMPORTANT]
```

is not part of the standard Markdown specification. It is interpreted as having a special meaning.

Therefore,

```md
> [!IMPORTANT]
> This is important information.
```

is a **GitHub-specific extension syntax** that utilizes Markdown's quote syntax.

As a result, while it may be displayed as a note box on GitHub, it may be displayed as a simple quote in other Markdown renderers.

## Alert Types You Can Use

GitHub mainly supports the following five types of Alerts:

### NOTE

Use this to indicate supplementary information.

```md
> [!NOTE]
> This setting is optional.
```

### TIP

Use this to indicate convenient methods or recommended ways of doing things.

```md
> [!TIP]
> You can easily check the settings using this command.
```

### IMPORTANT

Use this for important information that users should definitely know.

```md
> [!IMPORTANT]
> Back up your configuration file before upgrading.
```

### WARNING

Use this to alert users to operations that may cause problems.

```md
> [!WARNING]
> This operation will overwrite existing settings.
```

### CAUTION

Use this for operations that involve particularly serious risks, such as data loss.

```md
> [!CAUTION]
> Running this command will delete saved data.
```

## Writing Multiple Lines

If you want to write multiple lines of text within an Alert, add a `>` to each line.

```md
> [!IMPORTANT]
> The configuration method has changed.
>
> Before upgrading,
> check the new configuration method.
```

If you want to insert a blank line, write:

```md
>
```

This is the same mechanism as a standard Markdown blockquote.

## You Can Also Use Bold Text and Links

You can also use standard Markdown syntax within Alerts.

```md
> [!IMPORTANT]
> **Check before upgrading**
>
> See the [Migration Guide](docs/migration.md) for details.
```

This is useful when you want to guide readers to important documents in the README.

## When Should You Use It?

Alerts are useful for making the text in a README easier to read and organize.

For example:

- Notes for upgrading
- Breaking changes
- Security notes
- Configuration notes
- Common mistakes
- Recommended settings

In particular,

```md
## Upgrade

This describes how to upgrade.

> [!IMPORTANT]
> The configuration file format has changed in version 2 and later.

Run the following command.
```

Using it this way clearly separates the main text from important information.

## Be Careful Not to Overuse It

Although it is a useful syntax, if you use Alerts for all information, you will end up making it difficult to distinguish the importance of each item.

For example, if a README is filled with:

```md
> [!NOTE]
> ...

> [!TIP]
> ...

> [!IMPORTANT]
> ...

> [!WARNING]
> ...
```

like a box full of notes, it will be difficult to determine what to read.

Basically, use regular text, and use Alerts only for information you really want to highlight.

## The Display May Change Outside of GitHub

Another thing to keep in mind when using this syntax is the environment in which it will be displayed.

On GitHub,

```md
> [!IMPORTANT]
> Important information
```

is displayed as a dedicated Alert.

However, in Markdown renderers that do not support this syntax, it may be displayed as a regular quote:

```text
[!IMPORTANT]
Important information
```

Therefore, while this syntax is easy to use for documents intended to be read on GitHub, such as GitHub READMEs, you need to be aware of compatibility when displaying documents in various Markdown processing systems.

## Summary

The syntax you see on GitHub, such as

```md
> [!IMPORTANT]
```

is a Markdown extension called **Alerts**.

The `>` is the standard Markdown quote syntax, but the part that interprets `[!IMPORTANT]` and so on as a special note box is not included in standard Markdown.

On GitHub READMEs and Issues, it is a useful way to visually highlight important information.

The types you can use are:

```text
NOTE
TIP
IMPORTANT
WARNING
CAUTION
```

When writing important notes in a README, using this Alerts syntax can convey the priority of information more clearly than just using bold text.
