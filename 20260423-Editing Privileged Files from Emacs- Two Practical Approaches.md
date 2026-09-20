---
title: "Editing Privileged Files from Emacs: Two Practical Approaches"
description: "If you use Emacs with a user-owned daemon and emacsclient, editing privileged files can become..."
pubDatetime: 2026-04-23T07:47:25.770Z
---

If you use Emacs with a user-owned daemon and `emacsclient`, editing privileged files can become awkward.

At first glance, `sudo -e` (`sudoedit`) looks like the right tool. It is safer than running your editor directly as root, and it integrates cleanly with the usual `EDITOR`/`VISUAL` environment variables. But recent versions of `sudo` are deliberately conservative. In particular, `sudoedit` may refuse to edit a file if it lives in a directory that is writable by the calling user. That behavior comes from the `sudoers` policy option `sudoedit_checkdir`.

In practice, there are two reasonable ways to deal with this:

1. Adjust `sudoers` so that `sudoedit` allows the workflow you want.
2. Skip `sudoedit` entirely and open the file through Emacs TRAMP with `/sudo::`.

This article walks through both approaches and explains when each one makes sense.

---

## Why `sudoedit` blocks some files

`sudoedit` does not normally run your editor as root on the target file itself. Instead, it copies the file to a temporary location, launches your editor as your normal user, and then copies the modified contents back into place afterward.

That design is intentional. It reduces the risk of running a full interactive editor with elevated privileges.

On top of that, `sudoedit` includes some additional safety checks. One of them is controlled by `sudoedit_checkdir`. When that option is enabled, `sudoedit` refuses to edit files in directories that are writable by the invoking user. The point is to make certain classes of path manipulation and link attacks harder.

That is why a command that looks perfectly reasonable can still fail:

```bash
sudoedit some/file
```

If the parent directory is writable by you, `sudoedit` may reject it even though you technically have permission to create or modify files there.

---

## Approach 1: Change `sudoers` and keep using `sudoedit`

If you want to preserve the `sudoedit` model, the direct fix is to adjust `sudoers`.

### The setting that matters

The relevant option is:

```sudoers
Defaults !sudoedit_checkdir
```

Disabling it tells `sudoedit` not to reject files merely because they are under a user-writable directory.

### How to edit `sudoers` safely

Do not edit `/etc/sudoers` directly in a normal editor. Use `visudo`, which performs syntax checking and helps prevent lockouts caused by malformed configuration.

For a system-wide change:

```bash
sudo visudo
```

Then add:

```sudoers
Defaults !sudoedit_checkdir
```

If you want the change to apply only to a specific user:

```sudoers
Defaults:yourname !sudoedit_checkdir
```

A cleaner pattern is to place the override in a separate file under `/etc/sudoers.d/`:

```bash
sudo visudo -f /etc/sudoers.d/sudoedit
```

Then add:

```sudoers
Defaults:yourname !sudoedit_checkdir
```

That keeps the main `sudoers` file simpler and makes the change easier to audit later.

### Using `sudoedit` with `emacsclient`

Once `sudoedit` is allowed to do what you want, it works fine with a user-owned Emacs daemon.

For a graphical frame:

```bash
export SUDO_EDITOR='emacsclient -c'
sudoedit /etc/hosts
```

For terminal use:

```bash
export SUDO_EDITOR='emacsclient -t'
sudoedit /etc/hosts
```

This is often the cleanest setup if you like the `sudoedit` model and want to keep using it.

### Pros

* Preserves the intended `sudoedit` workflow
* Works naturally with `emacsclient`
* Keeps the editor itself running as your normal user

### Cons

* Weakens one of `sudoedit`’s built-in safety checks
* Requires administrative access to change policy
* Affects host policy, not just your personal workflow

---

## Approach 2: Use Emacs TRAMP with `/sudo::`

If your real goal is simply “open this file from my existing Emacs daemon, with privilege escalation when needed,” then TRAMP is often the more practical answer.

You can open a privileged local file like this:

```bash
emacsclient /sudo::/etc/hosts
```

Or from inside Emacs:

```text
C-x C-f /sudo::/etc/hosts
```

This uses TRAMP’s `sudo` method. From the user’s perspective, it feels like opening a normal file, except TRAMP arranges access through `sudo`.

### Why this avoids the `sudoedit` problem

Because this is not `sudoedit`.

That distinction matters. `sudoedit_checkdir` is a `sudoedit`-specific policy control. TRAMP `/sudo::` uses `sudo`, not `sudoedit`, so it is not subject to that particular check.

This does **not** mean TRAMP bypasses `sudoers` entirely. It still relies on `sudo`, so normal authentication and authorization rules still apply. If your `sudoers` configuration forbids the underlying access, TRAMP will not magically override that.

But if your problem is specifically the extra `sudoedit` directory check, `/sudo::` is a practical way around it.

### Pros

* No need to modify `sudoers`
* Fits naturally into an Emacs daemon + `emacsclient` workflow
* Avoids `sudoedit_checkdir`
* Feels seamless once you are already living inside Emacs

### Cons

* This is not the same security model as `sudoedit`
* You are editing through TRAMP’s `sudo` method, not through a temporary copy-and-replace flow
* Some people prefer `sudoedit` precisely because it is more restrictive

---

## What about TRAMP `/sudoedit::`?

TRAMP also has a `sudoedit` method:

```bash
emacsclient /sudoedit::/etc/hosts
```

That is a different thing from `/sudo::`.

If `/sudo::` is the “just let me open the file with elevated access” option, `/sudoedit::` is the “stay closer to the `sudoedit` safety model” option. It is designed to behave more like `sudoedit`, which also means it is less attractive if your entire goal is to escape `sudoedit`-specific restrictions.

So the practical distinction is:

* Use `/sudo::` when you want convenience and a smooth Emacs workflow.
* Use `/sudoedit::` when you explicitly want `sudoedit`-style behavior inside TRAMP.

---

## Can you use relative paths with TRAMP?

Yes, but the answer is slightly subtle.

In Emacs, relative paths are resolved against `default-directory`. That is true for TRAMP too.

So if your current buffer or Dired session is already in:

```text
/sudo::/etc/
```

then opening:

```text
hosts
```

will resolve to:

```text
/sudo::/etc/hosts
```

In other words, relative paths work perfectly well once your current directory is already a TRAMP path.

A common pattern is:

```text
C-x C-f /sudo::/etc/
```

and then from there, open files such as `hosts`, `fstab`, or `ssh/sshd_config` relative to that directory.

From a shell, though, it is usually simpler to pass a full TRAMP path to `emacsclient` rather than relying on implicit context.

---

## Which approach should you choose?

The answer depends on what you are optimizing for.

### Choose `sudoers` + `sudoedit` if:

* you want to keep the `sudoedit` workflow
* you prefer its temporary-file model
* you are comfortable changing system policy
* you want `emacsclient` to act purely as the editor frontend for `sudoedit`

### Choose TRAMP `/sudo::` if:

* you already use a user-owned `emacs -daemon`
* you want `emacsclient` to open privileged files directly and naturally
* you do not want to touch `sudoers`
* your main frustration is specifically `sudoedit_checkdir`

In day-to-day Emacs usage, `/sudo::` is usually the more ergonomic solution. It integrates better with how Emacs users actually work. By contrast, changing `sudoers` is more appropriate when you explicitly want to preserve the `sudoedit` model across the machine.

---

## A minimal practical setup

If you want the simplest workable arrangement, this is often enough:

```bash
alias ec='emacsclient -c'
```

Then open privileged files through TRAMP when needed:

```bash
ec /sudo::/etc/hosts
```

If you still want to keep `sudoedit` available as well:

```bash
export SUDO_EDITOR='emacsclient -c'
```

That gives you both options:

* `sudoedit /path/to/file` when you want the `sudoedit` model
* `emacsclient /sudo::/path/to/file` when you want the most direct Emacs workflow

---

## Final thoughts

There is no single universally correct answer here, because the two approaches serve slightly different goals.

Changing `sudoers` is the right move if you want to preserve `sudoedit` and simply relax one of its checks.

Using TRAMP `/sudo::` is the right move if your real priority is editing privileged files comfortably from an existing Emacs daemon without fighting `sudoedit`’s policy decisions.

If you are an Emacs-heavy user, the second option is often the one that feels more natural in practice. If you are managing system policy for a shared host, the first option may be the more disciplined choice.
