---
title: "How to Remove Specific Files from Git Commit History"
description: "Remove selected file changes from past commits with interactive rebase, choosing between dropping whole commits and amending only the affected file."
pubDatetime: 2026-09-01T05:52:58.212Z
updatedDate: 2026-09-01T07:45:26.128Z
---

When working with Git, you might encounter a situation where you want to:

> "Make it as if this file was never committed in the first place."

If you simply delete the file from the current branch, you can use `git rm` and commit. However, this only removes the file from the current state; it remains in the past commit history.

This article introduces a method using **interactive rebase (`git rebase --committer-date-is-author-date　-i`) to remove changes related to a specific file from past commits.**

Please note that this method rewrites the commit history. Be cautious when using it on branches that have already been pushed to a remote repository or are shared among multiple people.

## 1. Find the Commits Related to the Target File

First, identify the commits in which the file you want to remove was added or modified.

If you know the file path, run the following command:

```bash
git log --oneline --name-status -- path/to/file
```

If you don't know the exact name or path of the target file, you can check the commits and the files that were changed together:

```bash
git log --oneline --name-status
```

The important thing here is the status displayed on the left side of the file.

```text
A    path/to/file
M    path/to/file
```

You mainly need to check the following two:

* `A`: The file was newly added.
* `M`: An existing file was modified.

The approach will differ depending on this distinction.

## 2. Rebase from the Oldest Commit Related to the Target File

After identifying the commits related to the target file, find the **oldest commit** among them.

If that commit is `earliest_commit_id`, start the interactive rebase as follows:

```bash
git rebase --committer-date-is-author-date -i {oldest_commit_id}^
```

The `^` is added to include the target commit itself in the rebase range.

For example, if the oldest commit is `abc1234`, then:

```bash
git rebase --committer-date-is-author-date　-i abc1234^
```

## 3. Change the Target Commit to `drop` or `edit`

When you run the command, an editor will open and display the list of commits.

```text
pick abc1234 add config
pick def5678 update config
pick ghi9012 implement feature
```

From here, you will modify the commits related to the target file.

### If the Commit Only Changes That File

Since the commit itself is unnecessary, change `pick` to `drop`.

```text
drop abc1234 add config
```

This will remove the commit itself from the history.

However, **do not use `drop` if the commit contains changes you want to keep.**

### If the Commit Includes Changes to Other Files

Since you want to keep the changes to other files, you cannot delete the commit itself.

In this case, change `pick` to `edit`.

```text
edit abc1234 implement feature
```

When you start the rebase, the process will temporarily stop at that commit.

At this point, remove only the changes related to the target file.

## 4. Remove Changes to the Target File from the `edit` Commit

From here, the operation changes depending on whether the target file was **newly added or an existing file was modified** in that commit.

### If the File Was Newly Added

If the file was newly added in that commit, delete the file.

```bash
git rm path/to/file
```

Then, recreate the commit.

```bash
git commit --amend
```

Modify the commit message as needed.

### If an Existing File Was Modified

If the target file was modified in that commit, revert the target file to its state in the commit **immediately before** that commit.

```bash
git restore --source={commit_id}^ --staged --worktree -- path/to/file
```

For example, if the target commit is `abc1234`, then:

```bash
git restore --source=abc1234^ --staged --worktree -- path/to/file
```

This will remove the changes to the target file that were included in that commit.

Next, modify the commit.

```bash
git commit --amend
```

If there are any descriptions about the target file in the commit message, modify them here as appropriate.

## 5. Continue the Rebase

After finishing the commit modification, continue the rebase.

```bash
git rebase --continue
```

If there are multiple commits that you changed to `edit`, the process will stop again at the next `edit`.

Each time, repeat the following:

1. Remove the changes to the target file.
2. `git commit --amend`.
3. `git rebase --continue`.

If you complete all the steps, the rebase will be finished.

## Finally, Check the History

After the rebase is complete, confirm that the history related to the target file is as intended.

```bash
git log --name-status -- path/to/file
```

Also, check:

```bash
git status
```

and

```bash
git log --oneline
```

to be sure about the overall state of the branch.

## Caution: If You Have Already Pushed

`git rebase --committer-date-is-author-date　-i` is an operation that rewrites past commits.

Therefore, if you have already pushed the target commit to a remote repository, the commit IDs will be different between the local and remote repositories after the rebase.

If you need to reflect the rewritten history in the remote repository, you will need to perform a force push.

```bash
git push --force-with-lease
```

It is safer to use `--force-with-lease` instead of `--force`, as it is less likely to overwrite changes that others have added to the remote repository.

However, rewriting the history of a shared branch will affect other developers. If you are working on a branch that is used by a team, be sure to check the scope of the impact before running it.

## Summary

If you want to remove changes related to a specific file from the commit history, follow these steps:

```text
Find the commits related to the target file.
        ↓
Start git rebase --committer-date-is-author-date -i from the oldest commit.
        ↓
If the commit is unnecessary, use drop.
        ↓
If it includes other changes, use edit.
        ↓
Remove only the changes to the target file.
        ↓
git commit --amend.
        ↓
git rebase --continue.
```

The key is to use `drop` if you can delete the commit, and `edit` if you only want to delete part of the commit.

Also, if using `edit`, check whether the target file was "newly added" or "modified to an existing file" in that commit before proceeding.

Using interactive rebase, you can remove changes related to a specific file from past commits while keeping other changes.
