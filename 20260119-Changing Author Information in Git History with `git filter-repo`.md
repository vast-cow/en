---
title: "Changing Author Information in Git History with `git filter-repo`"
description: "Sometimes, you may need to update the author or committer information in your Git repository history...."
pubDatetime: 2026-01-19T15:28:22.616Z
updatedDate: 2026-01-23T13:22:12.450Z
---

Sometimes, you may need to update the author or committer information in your Git repository history. For example, if your email address has changed, or you want to fix a name typo in past commits. The `git filter-repo` tool makes this process easy and efficient.

Here is a command that rewrites all commits to use a new name and email:

```bash
git filter-repo --force --commit-callback $'new_name=b"New Name"\nnew_email=b"new@example.com"\nif (commit.author_name==new_name and commit.author_email==new_email and commit.committer_name==new_name and commit.committer_email==new_email):\n    return\ncommit.author_name=new_name\ncommit.author_email=new_email\ncommit.committer_name=new_name\ncommit.committer_email=new_email'
```

## Install

```bash
pip install git-filter-repo
```

## How it works:

* **`commit.author_name` and `commit.author_email`**: update the original author of each commit.
* **`commit.committer_name` and `commit.committer_email`**: update the person who committed the change (often the same as the author).

This command ensures that every commit in your repository history shows the new name and email.

## Important Notes:

1. **Backup first**: Rewriting history changes commit IDs. It’s a good idea to back up your repository before running this command.
2. **Force push required**: After rewriting history, you will need to push your changes with `git push --force` if the repository is shared.
3. **Coordinate with your team**: Since commit hashes change, other contributors will need to re-clone or reset their local repositories.

By using `git filter-repo`, you can clean up your project’s commit history and make sure your author details are consistent.
