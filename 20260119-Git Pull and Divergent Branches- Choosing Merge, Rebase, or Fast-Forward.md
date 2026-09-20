---
title: "Git Pull and Divergent Branches: Choosing Merge, Rebase, or Fast-Forward"
description: "If you’ve recently run:    git pull        Enter fullscreen mode            Exit fullscreen mode    ..."
pubDatetime: 2026-01-19T09:53:29.876Z
---

If you’ve recently run:

```bash
git pull
```

…and Git responded with a message like:

> Pulling without specifying how to reconcile divergent branches is discouraged.

you’re not alone. Newer versions of Git are nudging users toward an explicit choice when local and remote branches have **diverged**.

This post explains what that warning means, why Git is asking, and how to pick the right pull strategy for your workflow.

## What “Divergent Branches” Actually Means

A branch is considered *divergent* when:

* the remote branch has commits you don’t have locally, **and**
* your local branch has commits the remote doesn’t have.

In other words, both sides moved forward independently. When you pull, Git needs to decide how to integrate those histories.

Historically, Git defaulted to a merge-based pull in many environments. But because teams have different expectations for commit history (linear vs. explicit merges), Git now encourages you to make that preference explicit.

## `git pull` Is Two Commands in One

It helps to remember that `git pull` is essentially:

1. `git fetch` (download remote changes)
2. a second step to integrate changes (merge, rebase, or fast-forward)

The warning is about step (2): the integration strategy.

## The Three Strategies Git Offers

### 1) Merge: Preserve History as It Happened

A merge-based pull integrates the remote branch by creating a merge commit when necessary.

**Pros**

* Doesn’t rewrite history
* Clear, explicit record of integration points
* Lowest risk for shared branches like `main`

**Cons**

* Can create extra “merge commits”
* History can look busy in repos with frequent pulls

**Set merge as default (repo or global):**

```bash
git config pull.rebase false
# or
git config --global pull.rebase false
```

---

### 2) Rebase: Keep a Linear, Cleaner History

A rebase-based pull takes your local commits and replays them on top of the newly fetched remote branch.

**Pros**

* Linear, tidy commit history
* Often preferred for feature-branch workflows
* Makes `git log` easier to read

**Cons**

* Rewrites local commit hashes
* Requires more care if local commits were already shared

**Set rebase as default (repo or global):**

```bash
git config pull.rebase true
# or
git config --global pull.rebase true
```

**Optional: automatically stash uncommitted changes during rebase**

```bash
git config --global rebase.autoStash true
```

---

### 3) Fast-Forward Only: Refuse Divergence

Fast-forward only means: “pull only if I can move my branch pointer forward without any merges or rebases.”

If the histories diverged, Git will stop and ask you to resolve it manually.

**Pros**

* Prevents accidental merge commits
* Forces explicit handling of divergence
* Very safe in strict workflows

**Cons**

* You’ll hit errors more often
* You must resolve divergence manually (merge or rebase yourself)

**Set fast-forward only as default:**

```bash
git config pull.ff only
# or
git config --global pull.ff only
```

---

## One-Time Overrides (No Config Change)

If you don’t want to commit to a default yet:

```bash
git pull --no-rebase   # merge
git pull --rebase      # rebase
git pull --ff-only     # fast-forward only
```

This is useful when you’re in a mixed environment (e.g., rebase on feature branches, merge on main).

## Which One Should You Choose?

Here are pragmatic recommendations that work well in most teams:

* **Shared long-lived branches (main/develop):** use **merge**

  * lowest surprise, no history rewriting
* **Feature branches / personal development flow:** use **rebase**

  * cleaner history, easier review, fewer merge commits
* **Highly controlled repositories:** use **fast-forward only**

  * prevents accidental history shape changes

If you’re unsure, start with **rebase** for feature work and **merge** for mainline branches. Many teams effectively do this—either via developer convention or branch-specific tooling.

## Final Thoughts

Git’s warning isn’t an error; it’s a prompt. It’s Git reminding you that commit history is part of your project’s engineering culture.

Pick the strategy that matches your team’s expectations, set it once, and move on with confidence.
