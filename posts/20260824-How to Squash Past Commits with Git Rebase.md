---
title: "How to Squash Past Commits with Git Rebase"
description: "Combine commits through interactive rebase, handle conflicts and commit messages, and use --root when the initial commit must be included."
pubDatetime: 2026-08-24T08:20:38.714Z
---

Squashing commits with Git’s `rebase` command is useful when you want to clean up your commit history. Here’s a step-by-step guide.

---

## 1. Check Which Commits to Squash

First, review the commit log to decide which commits you want to combine:

```bash
git log --oneline
```

Example:

```plaintext
a1b2c3d Fix typo
d4e5f6g Add feature X
h7i8j9k WIP: implement feature X
...
```

Suppose you want to squash `Add feature X` and `WIP: implement feature X`.

---

## 2. Start an Interactive Rebase

Run `git rebase -i` using the commit **before** the ones you want to squash as the base:

```bash
git rebase -i <base_commit_id>
```

In this case, if `a1b2c3d` is the commit before, you would run:

```bash
git rebase -i a1b2c3d
```

---

## 3. Edit the Commit List

Your editor will open with something like this:

```plaintext
pick d4e5f6g Add feature X
pick h7i8j9k WIP: implement feature X
```

Change the second `pick` to `squash` (or just `s`):

```plaintext
pick d4e5f6g Add feature X
squash h7i8j9k WIP: implement feature X
```

---

## 4. Edit the Commit Message

Next, Git will prompt you to combine commit messages:

```plaintext
# This is a combination of 2 commits.
# The first commit’s message is:
Add feature X

# The following commit message will also be included:
WIP: implement feature X
```

Clean it up so the final message is concise:

```plaintext
Add feature X
```

---

## 5. Finish the Rebase

Save and close the editor. Git will apply the rebase and squash the commits into one.

---

## 6. Troubleshooting

* **If you hit conflicts**: resolve them as usual, then run:

  ```bash
  git add <fixed_files>
  git rebase --continue
  ```

* **If you want to cancel the rebase**:

  ```bash
  git rebase --abort
  ```

* **After rewriting history**: if the branch is pushed to a remote you will need to force-push the rewritten history:

  ```bash
  git push --force-with-lease
  ```

  (Prefer `--force-with-lease` over `--force` when collaborating.)

---

## 7. Squashing Including the Root Commit (`--root`)

By default `git rebase -i <base>` rebases starting **after** the specified base commit. If you want to include the very first (root) commit in an interactive rebase — for example to edit or squash the initial commit together with later commits — use the `--root` option:

```bash
git rebase -i --root
```

**What this does**

* Lists the root (initial) commit as the first entry in the interactive todo.
* Lets you `pick`, `reword`, `edit`, `squash`, or `fixup` the root commit just like any other commit.
* Rewrites the root and all descendant commits (so every commit ID in the branch will change).

**Common uses & examples**

1. *Squash all commits into a single initial commit*
   If you want a single commit that contains the whole history, keep the root as `pick` and mark every other commit as `squash` (or `s`):

   ```plaintext
   pick a1b2c3d Initial commit
   squash d4e5f6g Add feature X
   squash h7i8j9k WIP: implement feature X
   ...
   ```

   After saving, Git will open the combined commit-message editor so you can craft the final message.

2. *Edit or reword the root commit message*
   Change the first line to `reword` (or `r`) to update the initial commit message during the rebase:

   ```plaintext
   reword a1b2c3d Initial commit
   pick d4e5f6g Add feature X
   ...
   ```

   The rebase will stop and prompt you to edit the root commit message.

3. *Amend the root content*
   Use `edit` on the root entry if you need to change the actual tree of the root commit (add/remove files). Rebase will stop at that commit; then you can modify files, `git add` them, and run `git commit --amend` before continuing the rebase:

   ```bash
   # interactive todo:
   edit a1b2c3d Initial commit
   pick d4e5f6g Add feature X
   ...
   # when rebase stops:
   # make changes to the working tree
   git add <files>
   git commit --amend --no-edit   # or edit message as needed
   git rebase --continue
   ```

**Caveats**

* Rewriting the root rewrites the entire branch history. This is disruptive if the branch is shared — coordinate with collaborators or avoid rewriting published branches.
* After a `--root` rebase you must force-push to update any remote branch (`git push --force-with-lease`).
* `git rebase -i --root` behaves like a normal interactive rebase in terms of conflict resolution and aborting (`git rebase --abort`).

---

## Summary

* Use `git rebase -i <base>` to interactively squash commits after `<base>`.
* Mark commits to squash with `squash` (or `s`).
* Edit the commit message when prompted.
* Complete the rebase and resolve conflicts with `git rebase --continue`.
* To include the initial/root commit in your rebase (so you can squash or edit it), run `git rebase -i --root`. Be aware this rewrites the whole branch history and requires a force-push for published branches.
