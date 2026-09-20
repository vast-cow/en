---
title: "Rewriting a Specific String in a Specific File with git-filter-repo While Minimizing Changed Commits"
description: "This note explains a practical way to rewrite Git history using git-filter-repo to replace a..."
pubDatetime: 2026-01-25T14:01:42.300Z
---

This note explains a practical way to rewrite Git history using `git-filter-repo` to replace a particular string in a single target file, while keeping the number of rewritten commits as small as possible. The approach is implemented as a Bash script that identifies the earliest commit where the file was introduced, and then limits the history rewrite to only the commits from that point onward.

## Overview of the Approach

When you run a history rewrite across an entire repository, every commit can be rewritten, which is disruptive and can make review and coordination harder. This script reduces the blast radius by:

1. Finding the first commit where the target file was added.
2. Determining the commit range that starts just before that addition.
3. Running `git filter-repo` only on the relevant range (or the whole branch if no boundary exists).
4. Replacing a specific string with another only in the specified file, and only when it is safe and meaningful to do so.

## Script Configuration

At the top of the script, a few variables define what will be rewritten:

* `BRANCH=main`: the branch to rewrite.
* `FILE_PATH='HOGE'`: the path of the file to target (replace this with the real path).
* `FROM='fromtext'`: the literal text to search for.
* `TO='totext'`: the replacement text.

The `set -xeuo pipefail` line enables strict Bash behavior:

* `-e`: exit on error,
* `-x`: echo commands as they run,
* `-u`: treat unset variables as errors,
* `pipefail`: propagate failures in pipelines.

## Narrowing the Rewrite Range to Minimize Impact

A key feature is restricting the rewrite to commits that could possibly be affected.

### Finding the First Introduction of the File

The script computes:

* `OLDEST_COMMIT_ID`: the earliest commit where the target file was *added* (diff-filter `A`).

It does this by scanning the log for the file and selecting the first addition commit.

### Computing a Boundary Commit

Next, the script attempts to compute:

* `BOUNDARY_ID = OLDEST_COMMIT_ID^` (the parent of the “file added” commit)

This step can fail if the file was added in the root commit (which has no parent). The script temporarily disables `-e` (`set +e`) to capture the exit code and then restores strict mode.

### Building the `--refs` Range

If the boundary exists, the script sets:

* `REFS=(--refs "${BOUNDARY_ID}..${BRANCH}")`

This tells `git filter-repo` to rewrite only commits reachable in that range. If the boundary does not exist (root commit case), it falls back to rewriting the branch without a range restriction.

## Defining Literal Replacement Rules

`git filter-repo` supports text replacement rules via a file. The script creates a temporary rules file:

* `RULES="$(mktemp)"`
* It writes a single rule in literal mode:

  `literal:FROM==>TO`

Using literal mode avoids regex interpretation, which is safer when the search string contains special characters.

## Performing a Safe, File-Scoped Rewrite

The main rewrite is executed with:

* `git filter-repo --force`
* Optional `--refs ...` restriction (if computed)
* `--replace-text "$RULES"` to apply the replacement rules
* `--file-info-callback` to precisely control which blobs are rewritten

### How the Callback Minimizes Unnecessary Changes

The callback is written in Python (as supported by `git filter-repo`) and applies several guardrails:

1. **Only touch the target file**
   If the filename is not the specified `FILE_PATH`, the callback returns the original `(filename, mode, blob_id)` unchanged.

2. **Skip binary files**
   The callback loads the blob contents and checks `value.is_binary(contents)`. If binary, it returns unchanged to avoid corrupting non-text data.

3. **Only rewrite if a real change occurs**
   It applies the replace-text rules via `value.apply_replace_text(contents)`.
   If the result is identical to the original, it returns unchanged—this is important because it prevents rewriting commits where the target string does not appear.

4. **Create a new blob only when needed**
   If the content changes, it inserts a new blob (`value.insert_file_with_contents(new_contents)`) and returns that new blob ID.

These checks ensure that only commits which truly require changes (i.e., where the specified text appears in that file) will be rewritten. In practice, this keeps the rewritten commit set closer to the minimum necessary.

## Cleanup

Finally, the script removes the temporary rules file:

* `rm -f "$RULES"`

This keeps the environment clean and avoids leaving behind generated files.

## Practical Notes

* `--force` is used because history rewriting is destructive; it allows the operation to proceed even if `git filter-repo` detects conditions that would normally require confirmation.
* After rewriting history, any collaborators will need to re-clone or carefully reconcile their local histories, and you will typically need to force-push the rewritten branch to the remote.
* The range restriction and “no-op” avoidance logic are the two main mechanisms that keep the number of changed commits as small as possible.


```bash
#!/usr/bin/env bash
set -xeuo pipefail

BRANCH=main
FILE_PATH='HOGE'
FROM='fromtext'
TO='totext'

OLDEST_COMMIT_ID="$(git log --diff-filter=A --format=%H --reverse -- "$FILE_PATH" | head -n 1)"

set +e
BOUNDARY_ID="$(git rev-parse "${OLDEST_COMMIT_ID}^")"
GRV_CODE=$?
set -e

if [[ "$GRV_CODE" -eq 0 ]]; then
  REFS=(--refs "${BOUNDARY_ID}..${BRANCH}")
else
  REFS=()
fi

RULES="$(mktemp)"
printf 'literal:%s==>%s\n' "$FROM" "$TO" > "$RULES"

git filter-repo --force \
  "${REFS[@]}" \
  --replace-text "$RULES" \
  --file-info-callback "\
if filename != b'${FILE_PATH}':
  return (filename, mode, blob_id)

contents = value.get_contents_by_identifier(blob_id)
if value.is_binary(contents):
  return (filename, mode, blob_id)

new_contents = value.apply_replace_text(contents)
if new_contents == contents:
  return (filename, mode, blob_id)

new_blob_id = value.insert_file_with_contents(new_contents)
return (filename, mode, new_blob_id)
"

rm -f "$RULES"
```
