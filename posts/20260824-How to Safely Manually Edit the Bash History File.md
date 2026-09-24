---
title: "How to Safely Manually Edit the Bash History File"
description: "Coordinate in-memory Bash histories before and after editing HISTFILE, using append and reload operations while accounting for concurrent-session limitations."
pubDatetime: 2026-08-24T13:52:31.081Z
---

Bash command history is normally stored in `$HISTFILE` (in many environments, `~/.bash_history`).

Sometimes you may want to edit this file directly in an editor, for example to delete a batch of unwanted history entries.

However, care is required in environments where multiple Bash sessions are open at the same time. If you simply edit `$HISTFILE`, your changes may later be overwritten by history held by another Bash session.

This article outlines a procedure for **editing the history file as safely as possible while keeping multiple Bash sessions open**.

## Bash History Exists Both in Memory and in a File

The first thing to understand is that Bash history is not managed solely through `$HISTFILE`.

Each running Bash process independently keeps its own history in memory.

For example, if Bash is running in three terminals, the situation looks like this:

```text
Bash A ── history in memory
Bash B ── history in memory
Bash C ── history in memory
              ↓
         $HISTFILE
```

Therefore, when directly editing `$HISTFILE`, you need to consider not only the file itself, but also **the history held by each Bash process**.

## First, Write Out the History from All Sessions and Disable History Recording

First, run the following in **every Bash session** that is currently open:

```bash
history -a
set +o history
```

`history -a` appends history entries from that Bash session that have not yet been saved to `$HISTFILE`.

In other words, by running it in every session, you can collect each session's unsaved history into the file:

```text
Bash A ─┐
Bash B ─┼─ history -a → $HISTFILE
Bash C ─┘
```

The next command,

```bash
set +o history
```

temporarily disables history recording in that Bash session.

The key point is to run this in **every session** as well.

## Edit the History File from One Session

Once you have run

```bash
history -a
set +o history
```

in every Bash session, edit `$HISTFILE` from one of them.

For example, with Emacs:

```bash
emacsclient -nw "$HISTFILE"
```

Of course, you can use any editor you like.

```bash
vim "$HISTFILE"
```

or

```bash
"${EDITOR:-vi}" "$HISTFILE"
```

will work as well.

Make the necessary changes, such as deleting unwanted history entries, and save the file.

## Reload the History in Every Session After Editing

Simply editing the file does not remove the **pre-edit history still held in memory** by Bash processes that are already running.

Therefore, run the following again in **every Bash session**:

```bash
history -c
history -r "$HISTFILE"
set -o history
```

Each command has the following meaning.

```bash
history -c
```

clears the history currently held in memory by that Bash process.

Next,

```bash
history -r "$HISTFILE"
```

loads the edited `$HISTFILE`.

Finally,

```bash
set -o history
```

re-enables history recording.

This puts the sessions into the following state:

```text
              edited $HISTFILE
                    │
          history -r│
          ┌─────────┼─────────┐
          ↓         ↓         ↓
       Bash A    Bash B    Bash C
```

All sessions will now operate using the edited history as their baseline.

## Summary of the Procedure

### 1. Run in every Bash session

```bash
history -a
set +o history
```

### 2. Edit the history in one Bash session

```bash
"${EDITOR:-vi}" "$HISTFILE"
```

### 3. Run in every Bash session

```bash
history -c
history -r "$HISTFILE"
set -o history
```

That is the basic procedure.

## Why Use `history -a` Instead of `history -w`?

At first glance, it may seem safer to run

```bash
history -w "$HISTFILE"
```

before editing.

However, caution is required when multiple Bash sessions exist.

`history -w` **rewrites `$HISTFILE` using the history currently held in memory by the current Bash process**.

For example, suppose you have:

```text
Bash A history: A1 A2 A3
Bash B history: B1 B2 B3
```

Even if Bash B's history has already been written to `$HISTFILE`, Bash A may not have loaded it.

If you then run

```bash
history -w "$HISTFILE"
```

from Bash A, the file will be rewritten based on the history known to Bash A, potentially causing history originating from other sessions to be lost.

By contrast,

```bash
history -a
```

**appends** unsaved history entries.

Therefore, when collecting history from multiple sessions,

```bash
history -a
```

is more appropriate.

## Even This Is Not "Perfect"

Following the procedure above makes the operation considerably safer, but strictly speaking, it is not foolproof.

The Bash history file is not a transactional database designed for concurrent updates from multiple Bash processes.

In other words, if a new command is executed in another session while you are running

```bash
history -a
```

in each session, or if writes to the history file overlap, timing-dependent problems can still occur.

Also, commands executed after

```bash
set +o history
```

will not be recorded in that session's normal history.

Therefore, it is best to avoid continuing normal work while editing the history.

## The Most Reliable Approach Is to Close the Other Bash Sessions

If reliability is the priority, there is a simpler approach.

**Close every Bash session except the one you will use to edit the history.**

The fact that multiple processes are accessing the same `$HISTFILE` is itself the source of potential conflicts, so reducing

```text
Bash A
Bash B
Bash C
```

to

```text
Bash A
```

before editing the history substantially reduces the number of issues you need to account for.

In other words:

* Want to keep multiple sessions open → write out the history from every session, disable recording, edit, and reload it in every session.
* Prioritize reliability → close the other Bash sessions before editing.

## Conclusion

If you want to edit `$HISTFILE` while keeping multiple Bash sessions open, the following procedure is practical.

**Before editing, in every session:**

```bash
history -a
set +o history
```

**In one session:**

```bash
"${EDITOR:-vi}" "$HISTFILE"
```

**After editing, in every session:**

```bash
history -c
history -r "$HISTFILE"
set -o history
```

The important point is not to stop after editing `$HISTFILE` itself.

Each Bash process independently keeps its own history in memory. Therefore, the underlying idea is to **collect each session's unsaved history into `$HISTFILE` before editing, and then synchronize every session with the contents of the edited `$HISTFILE` afterward**.

However, Bash history management does not provide strong mutual exclusion between multiple processes. If you need to edit the history with maximum reliability, the simplest approach is to close the other Bash sessions before doing so.
