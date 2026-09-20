---
title: "Alt-g and Readline shortcuts that make Bash completion a bit more convenient"
description: "Let's say you're using Bash and you have a directory like this:    $ ls aaa-bbb-ccc        Enter..."
pubDatetime: 2026-09-18T11:05:17.941Z
---

Let's say you're using Bash and you have a directory like this:

```bash
$ ls
aaa-bbb-ccc
```

You want to move to this directory, but you only remember `bbb` in the middle of the name, not the leading `aaa`.

You might be tempted to do:

```bash
cd bbb<TAB>
```

But normal Bash TAB completion is basically prefix-based, so this won't complete `aaa-bbb-ccc`.

That's where glob comes in.

```bash
cd *bbb*
```

Since `*` matches any string, `aaa-bbb-ccc` matches.

However, here you might want a bit more:

```bash
cd *bbb*
```

At the point you've typed this, you want it to expand to:

```bash
cd aaa-bbb-ccc/
```

so you can confirm before executing.

That's where **`Alt-g`** comes in handy.

## `Alt-g`: Complete using glob

Readline, which handles Bash's command-line editing, has a feature called `glob-complete-word`.

It's assigned to `Alt-g` by default, so if you type:

```bash
cd *bbb*
```

and press `Alt-g`, it can complete using names that match the glob.

For example:

```bash
$ cd *bbb*<Alt-g>
```

becomes:

```bash
$ cd aaa-bbb-ccc/
```

This is quite handy when you only remember part of a name.

If you use it alongside normal TAB completion:

```text
cd aaa<TAB>       # when you remember the beginning
cd *bbb*<Alt-g>   # when you only remember the middle
```

## `Ctrl-x *`: Expand glob in place

Something worth remembering alongside `Alt-g` is:

```text
Ctrl-x *
```

This is `glob-expand-word`, which expands the glob pattern itself into the matched filenames.

For example:

```bash
$ echo *bbb*
```

If you press `Ctrl-x *`, it becomes:

```bash
$ echo aaa-bbb-ccc
```

If multiple names match, they all get expanded together.

A good way to think about it:

```text
Alt-g      → "complete" using glob
Ctrl-x *   → "expand" using glob
```

## `Alt-.`: Reuse the last argument of the previous command

One shortcut I personally recommend remembering if you use Bash is `Alt-.`.

For example:

```bash
mkdir /tmp/very-long-directory-name
```

Immediately after running that, if you do:

```bash
cd <Alt-.>
```

You get:

```bash
cd /tmp/very-long-directory-name
```

In other words, it inserts the last argument of the previous command.

This also works for creating a file and then immediately opening it:

```bash
touch some-very-long-filename.txt
vim <Alt-.>
```

If you press `Alt-.` repeatedly, you can go further back through the last arguments of older commands.

## `Ctrl-r`: Search command history

Many people probably already know this one, but `Ctrl-r` is also very useful.

Pressing it lets you incrementally search through past commands.

For example, if you previously ran:

```bash
docker compose exec app bundle exec rails console
```

and want to run it again, you don't need to scroll endlessly through history with the arrow keys.

Just press `Ctrl-r` and type:

```text
rails
```

and it will find history entries containing that.

This is essential if you frequently reuse long commands.

## `Ctrl-x Ctrl-e`: Edit the current command in an editor

When commands get long, editing on a single line in the shell can become painful.

In that case, press:

```text
Ctrl-x Ctrl-e
```

This lets you edit the command you're currently typing in `$EDITOR`.

For example, if you're in the middle of building a long command like:

```bash
find . -type f -name '*.log' -mtime +7 ...
```

you can press:

```text
Ctrl-x Ctrl-e
```

and edit it in your configured editor such as `vim` or `nano`.

When you save and exit, the content is returned to the shell and executed.

This is very handy for complex one-liners.

## Other useful Readline operations to remember

Bash uses Emacs-style Readline keybindings by default.

Here's a summary of commonly used ones:

| Key               | Action                           |
| ----------------- | -------------------------------- |
| `Alt-g`           | Complete using a glob pattern    |
| `Ctrl-x *`        | Expand glob into actual filenames |
| `Alt-*`           | Insert all completion candidates |
| `Alt-.`           | Insert last argument from previous command |
| `Ctrl-r`          | Search command history           |
| `Ctrl-x Ctrl-e`   | Edit current command in an editor |
| `Ctrl-a`          | Move to beginning of line        |
| `Ctrl-e`          | Move to end of line              |
| `Alt-b`           | Move back one word               |
| `Alt-f`           | Move forward one word            |
| `Ctrl-w`          | Delete word before cursor        |
| `Alt-d`           | Delete word after cursor         |
| `Ctrl-u`          | Delete before cursor             |
| `Ctrl-k`          | Delete after cursor              |
| `Ctrl-_`          | Undo                             |
| `Alt-#`           | Comment out current command and add to history |

You don't need to memorize all of them.

Just remembering:

```text
Alt-g
Alt-.
Ctrl-r
Ctrl-x Ctrl-e
Ctrl-x *
```

is enough to make Bash operations considerably easier.

## You can also assign TAB to glob completion

By the way, if you change the Readline settings, you can assign `glob-complete-word` (normally on `Alt-g`) to TAB.

In `~/.inputrc`, add:

```text
"\t": glob-complete-word
```

Then you can do:

```bash
cd *bbb*<TAB>
```

However, this replaces normal TAB completion.

Since it also affects everyday operations like:

```bash
cd aaa<TAB>
```

I personally prefer keeping TAB as normal completion and using:

```text
normal completion → TAB
glob completion   → Alt-g
```

as a division of roles, which I find easier to handle.

## Summary

The discovery here is simple:

For a directory like:

```bash
aaa-bbb-ccc/
```

if you type:

```bash
cd *bbb*
```

and press:

```text
Alt-g
```

you get glob-based completion.

Bash has quite a few Readline features that you won't easily notice if you only ever use TAB and arrow keys.

In particular, these five:

```text
Alt-g          glob completion
Ctrl-x *       glob expansion
Alt-.          reuse previous argument
Ctrl-r         history search
Ctrl-x Ctrl-e  edit in editor
```

are worth knowing and will help in everyday shell operations.

If you want to speed up your Bash workflow, beyond just adding commands and tools, looking into Readline keybindings is also recommended.
