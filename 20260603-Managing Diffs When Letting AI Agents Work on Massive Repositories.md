---
title: "Managing Diffs When Letting AI Agents Work on Massive Repositories"
description: "When you let an AI agent operate on a very large repository, managing the differences before and..."
pubDatetime: 2026-06-03T08:16:14.445Z
---

When you let an AI agent operate on a very large repository, managing the differences before and after the work is important. If you cannot tell which files the agent changed or which point you can roll back to, it becomes difficult to work safely.

Normally, Git is used for this kind of change management. However, when the target repository is extremely large, Git operations can become very slow. In projects with huge numbers of files or large-scale modifications, even checking status or reviewing diffs can take a significant amount of time.

## Rethinking the Problem When Git Becomes Heavy

The goal of running AI agents on massive repositories is not necessarily to create a detailed commit history. In many cases, what you really need is:

* A way to preserve the original state before work begins
* A way to see what changed afterward

If that is the objective, you do not have to rely exclusively on Git. An alternative approach is to use the file system's Copy-on-Write (CoW) functionality to create a working copy.

## Creating Copies with Copy-on-Write

File systems such as XFS can create fast copies using Copy-on-Write. Instead of physically duplicating every file immediately, the original data is shared until modifications actually occur.

As a result, even very large repositories can be copied quickly. You can let the AI agent work on the copied repository while preserving the original directory as a baseline for comparison.

## Common File Systems That Support CoW

To take advantage of Copy-on-Write, the underlying file system must support it. Common options include the following.

### XFS

XFS is widely used in Linux environments. When XFS is configured with reflink support, you can create fast CoW copies using `cp --reflink`.

For large repository workloads, XFS is often a practical choice. It integrates well into typical Linux server environments and allows working copies for AI agents to be created quickly.

### Btrfs

Btrfs is a Linux file system designed around Copy-on-Write principles. It provides features such as snapshots and subvolumes, making it well suited for preserving repository state before work begins.

By creating a snapshot before handing a directory to an AI agent, it becomes easier to inspect differences afterward or revert changes if necessary.

### APFS

APFS is the file system used by macOS. It supports Copy-on-Write and is useful when working with large directories on a Mac.

If you are letting an AI agent edit code locally on macOS, APFS can efficiently create working copies without duplicating all underlying data.

### ZFS

ZFS also provides Copy-on-Write capabilities. Its snapshot and cloning features are particularly powerful for preserving pre-change states while safely experimenting with modifications.

However, deployment and administration can be more complex depending on the environment. If you already use ZFS, it is a strong option.

## Typical Workflow

The workflow is straightforward:

1. Preserve the original repository.
2. Create a Copy-on-Write working copy.
3. Let the AI agent operate on the working copy.
4. Compare the working copy with the original directory after the work is complete.
5. Integrate only the desired changes.

On Linux systems using XFS or Btrfs, you can create a working copy with a command such as:

```bash
cp -a --reflink=always original-repo work-repo
```

This command creates a fast CoW copy rather than physically duplicating all data. The AI agent works in `work-repo`, while `original-repo` remains available as a reference point for comparison.

## When This Approach Works Well

This method is particularly useful when AI agents need to modify very large repositories. Examples include:

* Large-scale refactoring
* Mechanical code transformations
* Changes spanning thousands of files
* Automated migration tasks

On the other hand, if your goal is to manage normal development history, Git should remain the primary tool. Copy-on-Write copies are best viewed as a way to quickly provision AI workspaces and safely inspect changes afterward.

## Summary

In massive repositories, Git-based change management can become slow enough to reduce the efficiency of AI agents. In such cases, creating working copies with Copy-on-Write-capable file systems such as XFS, Btrfs, APFS, or ZFS can be an effective alternative.

Rather than repeatedly performing expensive Git operations, you can rapidly clone the repository state, allow the AI agent to modify the copy, and then compare the results against the original. This provides a lightweight and efficient way to manage changes in large codebases while maintaining a safe rollback point.
