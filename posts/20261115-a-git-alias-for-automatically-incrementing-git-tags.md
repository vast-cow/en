---
title: "Git alias to automatically increment Git tags"
description: "This article explains how to configure a Git alias that automatically detects existing tags and creates a new tag by incrementing the patch version."
pubDatetime: 2026-11-15T22:16:00+09:00
---
When using Git for version control, you may want to add tags like `v1.2.3`.

It can be a bit cumbersome to check existing tags, find the latest version, increment the patch version, and then create a new tag every time.

Therefore, let's create a Git alias called `git tag-bump` so that you can create the next version tag with just one command.

## What we want to achieve

For example, let's assume the following tags exist:

```text
v1.0.0
v1.2.3
v1.2.9
v1.2.10
```

If you run:

```bash
git tag-bump
```

in this state, it will increment the patch version of the latest `v1.2.10` and create `v1.2.11`.

## Registering the Git alias

Run the following command only once.

```bash
git config --global alias.tag-bump '!f() { git tag "$(git tag -l "v*.*.*" --sort=-version:refname | awk -F. '"'"'/^v[0-9]+\.[0-9]+\.[0-9]+$/ {print $1 "." $2 "." ($3+1); exit}'"'"')"; }; f'
```

The setup is now complete.

From now on, just run the following command in the target Git repository.

```bash
git tag-bump
```

You can check the created tags with the following command.

```bash
git tag -l 'v*.*.*' --sort=-version:refname
```

## How it works

The command you registered can be organized into the following shell script.

```bash
f() {
  git tag "$(
    git tag -l "v*.*.*" --sort=-version:refname |
    awk -F. '/^v[0-9]+\.[0-9]+\.[0-9]+$/ {
      print $1 "." $2 "." ($3+1);
      exit
    }'
  )"
}
f
```

Let's look at it step by step.

### 1. List version-formatted tags

```bash
git tag -l "v*.*.*"
```

Lists tags that match `v*.*.*`.

### 2. Sort in version order

```bash
git tag -l "v*.*.*" --sort=-version:refname
```

`version:refname` is a version-aware sorting method provided by Git.

The leading `-` sorts it in descending order.

```text
v1.2.10
v1.2.9
v1.2.3
v1.0.0
```

Normal string sorting would determine that `v1.2.9` is greater than `v1.2.10`, but version sorting takes the numerical values into account.

Note that `version:refname` is not a strict SemVer parser, but it is suitable for `vnumber.number.number` formats like the one we are using.

### 3. Increment the patch version

```bash
awk -F. '/^v[0-9]+\.[0-9]+\.[0-9]+$/ {
  print $1 "." $2 "." ($3+1);
  exit
}'
```

Uses `awk` to increment the patch part of the tag.

`-F.` specifies that `.` is to be used as the delimiter.

For example, for `v1.2.10`:

| Field | Value |
|---|---|
| `$1` | `v1` |
| `$2` | `2` |
| `$3` | `10` |

`$3+1` increments the patch, and outputs `v1.2.11`.

Also, because the regular expression targets only formats like `v1.2.3`, pre-release tags like `v1.2.3-rc.1` are excluded.

`exit` processes only the first valid tag found.

### 4. Create a new tag

Finally, the shell's command substitution `$(...)` is used to pass the generated version to `git tag`.

```bash
git tag "$( ... )"
```

This creates a new tag at the current `HEAD`.

## Why the `!` is added to the Git alias

Git aliases are usually used as abbreviations for Git subcommands.

For example, if you set:

```bash
git config --global alias.st status
```

then `git st` will execute `git status`.

On the other hand, adding a leading `!` executes it as a shell command instead of a Git subcommand.

This time, since we are using pipes, `awk`, and command substitution, we are using `!`.

Also, `'"'"'` in the registered command is a shell notation for including single quotes within a string enclosed in single quotes.

## Precautions

There are a few things to keep in mind when using this alias.

- **Existing tag is required:** The initial tag creation process is not included, so an error will occur if there are no tags to target.
- **Targeting local tags:** If you have not fetched remote tags, run `git fetch --tags` beforehand.
- **Creates a lightweight tag:** If an annotated tag is required, modification to `git tag -a` etc. is necessary.
- **Tag push is required separately:** The new tag is created locally, so you need to run `git push origin v1.2.11` to reflect it remotely.
- **Does not change major/minor:** It always increments only the patch part of the latest version.

## Summary

By combining Git's built-in `--sort=-version:refname` and `awk`, you can automate tag increment without relying on external version management tools.

Once you set up the Git alias, just use it like this:

```bash
git tag-bump
```

It's a small automation, but it's useful because it reduces the operations you repeat during release.
