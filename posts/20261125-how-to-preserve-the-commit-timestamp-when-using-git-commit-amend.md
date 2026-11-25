---
title: "How to preserve the commit date and time when using `git commit --amend`"
description: "This article explains how to use the `--amend` option while specifying the original commit date and time using the environment variable `GIT_COMMITTER_DATE`."
pubDatetime: 2026-11-25T22:58:00+09:00
---
If you don't want to change the commit date and time with `git commit --amend`, specify the date and time before the change in `GIT_COMMITTER_DATE`.

```
GIT_COMMITTER_DATE="$(git show -s --format=%cI HEAD)" git commit --amend
```

If you also don't want to change the commit message:

```
GIT_COMMITTER_DATE="$(git show -s --format=%cI HEAD)" git commit --amend --no-edit
```

## **Git Date and Time Differences**

| Date and Time          | Normal `--amend` | Above Command |
| --------------------- | ---------------- | ------------- |
| AuthorDate (Creation Date) | Preserved        | Preserved     |
| CommitDate (Commit Date)   | Updated          | Preserved     |

You can check both dates and times with the following command:

```
git log -1 --format=fuller
```

## **Register as an Alias (Recommended)**

If you use this frequently, it is convenient to register it as a Git alias.

```
git config --global alias.amend-keep-date '!f() { GIT_COMMITTER_DATE="$(git show -s --format=%cI HEAD)" git commit --amend "$@"; }; f'
```

After that, you can execute it with the following command:

```
git amend-keep-date
```

```
git amend-keep-date --no-edit
```

**Note:** Even if you preserve the date and time, the commit hash will change if you change the commit content or message.
