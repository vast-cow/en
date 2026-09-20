---
title: "Rewriting Remote Git History with `reset` and `push --force-with-lease`"
description: "Sometimes a commit is pushed to a remote repository by mistake, and the goal is to move the remote..."
pubDatetime: 2026-01-19T07:48:23.141Z
updatedDate: 2026-09-20T09:55:17.238Z
---

Sometimes a commit is pushed to a remote repository by mistake, and the goal is to move the remote branch back to an earlier point as if those commits never happened. In Git, this is done by rewriting history: you reset your local branch to the desired commit and then force-update the remote branch to match. Because rewriting history can disrupt other collaborators, it should be used with extra caution on shared branches.

## Overview: `reset` + force push

The common workflow is:

1. Check out the branch you want to fix.
2. Fetch the latest remote state to ensure you are working with current information.
3. Hard reset your local branch to the commit you want the branch to return to.
4. Push the rewritten branch back to the remote using `--force-with-lease`.

This approach replaces the remote branch tip with the commit you specify, effectively removing later commits from the branch’s visible history.

## Example: Remove the most recent commit(s) by returning to a specific commit

If you know the exact commit you want the branch to point to, you can reset to that commit and push the updated history:

```bash
git checkout <branch>
git fetch origin
git reset --hard <commit_sha_to_return_to>   # Return to that point in history
git push --force-with-lease origin <branch>
```

After this, the remote branch will point to `<commit_sha_to_return_to>`, and any commits that previously came after it will no longer be part of the branch history.

## Example: Remove the last N commits

If the mistake is simply the last few commits, you can reset relative to `HEAD`:

```bash
git reset --hard HEAD~N
git push --force-with-lease origin <branch>
```

Here, `N` is the number of most recent commits you want to remove.

## Why `--force-with-lease` is safer than `--force`

A standard force push (`--force`) overwrites the remote branch unconditionally, which can accidentally delete other people’s work if they pushed changes after you last updated your local view.

`--force-with-lease` adds a safety check: it refuses to update the remote branch if the remote has moved in the meantime (for example, if someone else pushed new commits). This makes it a safer default for history rewrites, especially when there is any chance the branch is being updated by others.

## Key caution for shared branches

Rewriting remote history changes the commit graph that others may have already based work on. If other developers have pulled the branch, they will need to reconcile their local history (often via rebase or reset) after the rewrite. For that reason, this method is best reserved for branches where the impact is understood and controlled.
