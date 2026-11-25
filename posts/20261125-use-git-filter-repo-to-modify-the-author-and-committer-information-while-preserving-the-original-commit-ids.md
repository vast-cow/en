---
title: "Using git filter-repo to modify the Author/Committer while preserving past commit IDs."
description: "This article explains how to use the `--refs` option in `git filter-repo` to limit the scope of the rewrite and how to use `--commit-callback` to modify the Author and Committer information."
pubDatetime: 2026-11-25T22:32:00+09:00
---
## Using `git filter-repo` to Correct Author and Committer Information in a Range of Commits

It can happen that you create multiple commits in Git without realizing that you have incorrectly set `user.name` or `user.email`.

To collectively correct the Author and Committer information recorded in past commits, `git filter-repo` is a useful tool.

However, instead of rewriting the entire repository's history, you want to rewrite only the necessary range and preserve the commit IDs of earlier commits.

In this example, we will use `git log` to identify the commits to be rewritten, and the `--refs` option of `git filter-repo` to limit the scope of the history rewrite.

## What You Want to Achieve

Suppose you have the following commit history:

```text
A --- B --- C --- D --- E --- F (main)
            *           *
```

Let's assume that `C` and `E` were created with incorrect user information.

After the rewrite, you want the history to look like this:

```text
A --- B --- C' -- D' -- E' -- F' (main)
```

- The commit IDs of `A` and `B` should remain unchanged.
- The Author/Committer of `C` and `E` should be corrected.
- The Author/Committer of `D` and `F` should not be changed.

Since Git's commit IDs also depend on the IDs of the parent commits, rewriting `C` will also change the commit IDs of its descendants, `D`, `E`, and `F`.

On the other hand, there is no need to rewrite the history before `C`.

## 1. Identify the Commits to be Rewritten

First, use `git log` to list the commits containing the incorrect user information.

In this example, we will correct the following user information:

| Item | Before | After |
|---|---|---|
| Name | Old Name | New Name |
| Email | old@example.com | new@example.com |

Extract the commits that match the Author or Committer name and email address.

```bash
git log main --reverse --topo-order \
  --format='%H%x09%an%x09%ae%x09%cn%x09%ce%x09%s' |
  awk -F '\t' \
    -v name='Old Name' \
    -v email='old@example.com' \
    '($2 == name && $3 == email) || ($4 == name && $5 == email)'
```

The following placeholders are used in the `git log` format specification:

| Placeholder | Description |
|---|---|
| `%H` | Commit ID |
| `%an` | Author name |
| `%ae` | Author email address |
| `%cn` | Committer name |
| `%ce` | Committer email address |
| `%s` | Commit message subject |
| `%x09` | Tab character |

By specifying `--reverse --topo-order`, parent commits are displayed before child commits.

If the commit history is linear, the first entry in the output will be the oldest of the commits to be rewritten.

Note that specifying `git log --author=... --committer=...` creates an AND condition, so we are using `awk` to extract the commits with an OR condition in this case.

## 2. Determine the Rewrite Range

Suppose the oldest commit ID to be rewritten is `abc1234`.

To rewrite this commit and everything up to the tip of `main`, you can specify the following range:

```bash
abc1234^..main
```

`A..B` represents the range of commits reachable from `B` but not reachable from `A`.

Therefore, to include `abc1234` itself, the `^` representing its parent commit is needed.

You can actually check the range to be rewritten with the following command:

```bash
git log --oneline abc1234^..main
```

Also, to prepare for post-rewrite verification, record the commit ID that forms the boundary and the current branch tip.

```bash
base=$(git rev-parse abc1234^)
old_tip=$(git rev-parse main)
```

## 3. Rewrite the History with `git filter-repo`

Rewriting history is a destructive operation, so it is recommended to run it in a newly cloned working repository, preserving the original repository.

Use `--refs` to limit the rewrite range and `--commit-callback` to modify the Author and Committer.

```bash
git filter-repo \
  --refs "$base..main" \
  --commit-callback '
    old_name = b"Old Name"
    old_email = b"old@example.com"

    new_name = b"New Name"
    new_email = b"new@example.com"

    if (commit.author_name == old_name and
        commit.author_email == old_email):
        commit.author_name = new_name
        commit.author_email = new_email

    if (commit.committer_name == old_name and
        commit.committer_email == old_email):
        commit.committer_name = new_name
        commit.committer_email = new_email
  '
```

Since Author and Committer are judged independently, if only one of them is incorrect, only that information will be corrected.

### The Role of `--refs`

Specifying `--refs` implicitly enables `--partial`, which performs a partial rewrite of the history.

By limiting the rewrite range as in this example, you can avoid regenerating commits before the range and preserve their commit IDs.

```text
Before rewrite:

A --- B --- C --- D --- E --- F
            |                 |
            +--- Rewrite range -+

After rewrite:

A --- B --- C' -- D' -- E' -- F'
```

Before rewriting, you can also add the `--dry-run` option to confirm.

## 4. Verify the Rewrite Results

First, check the Author and Committer information.

```bash
git log main \
  --format='%h | %an <%ae> | %cn <%ce> | %s'
```

Re-run the command from step 1 to confirm that no commits with the incorrect user information remain.

Next, verify that the commits before the rewrite range are still maintained as ancestors in the new history.

```bash
git merge-base --is-ancestor "$base" main \
  && echo "Original history preserved"
```

Also, check that there are no differences in the file contents, such as source code.

```bash
git diff --exit-code "$old_tip" main
```

In addition, `git filter-repo` generates a mapping of commit IDs before and after the rewrite.

```bash
cat .git/filter-repo/commit-map
```

By referring to this file, you can check which commit ID was changed to which commit ID.

## 5. Correct `user.name` and `user.email`

To prevent the same problem from occurring in the future, also correct the Git settings.

```bash
git config --local user.name "New Name"
git config --local user.email "new@example.com"
```

You can check the setting values and their sources with the following commands:

```bash
git config --show-origin --get user.name
git config --show-origin --get user.email
```

## Important Notes

- In histories including merges, simply using the parent of the first displayed commit as the boundary may exclude some of the commits to be corrected. You need to ensure that all commits to be corrected are included in the range.
- If the first commit is the one to be corrected, it will not have a parent commit, so you should specify it as `--refs main` or similar.
- Commits on other branches or tags that are not within the rewrite range may contain the history before the rewrite.
- Signed commits within the rewrite range may lose their signature.
- If you have already pushed to a remote, you will need to use force push to update the history. Be careful of the impact on other developers in shared branches.

## Summary

When correcting incorrect `user.name` or `user.email`, you do not need to rewrite the entire repository history.

The steps in this example are as follows:

1.  List the commits to be corrected using `git log`.
2.  Identify the oldest commit to be corrected.
3.  Limit the rewrite range with `--refs`.
4.  Correct the Author/Committer with `--commit-callback`.
5.  Verify the commit IDs and file contents before and after the rewrite.

The key is that **selecting the commits to be corrected and limiting the scope of the history rewrite are separate operations**.

By combining `--commit-callback` and `--refs`, you can correct only the necessary user information while preserving the IDs of older commits that do not need to be affected.
