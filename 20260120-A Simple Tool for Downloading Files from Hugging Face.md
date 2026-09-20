---
title: "A Simple Tool for Downloading Files from Hugging Face"
description: "Developers who work with machine learning models often need to download files from Hugging Face..."
pubDatetime: 2026-01-20T02:43:25.912Z
---

Developers who work with machine learning models often need to download files from Hugging Face repositories. While the Hugging Face website provides links, manually handling URLs and paths can be inconvenient. To make this process easier, the script `hf_get_from_url.py` was created.

## What the Script Does

The script helps users fetch files directly from Hugging Face by interpreting different types of inputs. It can handle:

* Full Hugging Face URLs with `blob` or `resolve`.
* Shortened forms like `huggingface.co/owner/repo/blob/main/file`.
* Simple repo and file paths, such as `owner/repo/file`.
* Direct repository references with or without file paths.

Once the input is parsed, the script uses the Hugging Face command-line interface (`hf`) to download the specified file into a local directory.

## Key Features

1. **Flexible Input Parsing**:
   The script can clean and interpret URLs or paths in different formats, ensuring users don’t need to worry about exact syntax.

2. **Dry Run Mode**:
   With the `--dry-run` option, users can preview the exact command that would be executed without actually downloading the file.

3. **Custom Local Directory**:
   By default, files are downloaded into a folder named after the repository. With the `--flatten-localdir` option, the folder name uses hyphens instead of slashes for convenience.

4. **Error Handling**:
   The script checks if the `hf` command is installed and provides helpful error messages if something goes wrong.

## Example Usage

```bash
# Download using a full URL
python hf_get_from_url.py "https://huggingface.co/owner/repo/blob/main/file.gguf"

# Download using a repo-style path
python hf_get_from_url.py owner/repo/file.gguf

# Preview the command without downloading
python hf_get_from_url.py --dry-run "huggingface.co/owner/repo/blob/main/file.gguf"
```

## Why It’s Useful

This tool simplifies the workflow for anyone who frequently downloads models, datasets, or configuration files from Hugging Face. By accepting different input styles and providing clear feedback, it makes the process faster and less error-prone.

```python
#!/usr/bin/env python3
"""
hf_get_from_url.py
Download files or directories from Hugging Face Hub using huggingface_hub API.

Usage:
  python hf_get_from_url.py [--dry-run] [--flatten-localdir] <input> [<input> ...]

Examples:
  python hf_get_from_url.py "https://huggingface.co/owner/repo/blob/main/path/to/file.gguf"
  python hf_get_from_url.py owner/repo/path/to/file.gguf
  python hf_get_from_url.py --dry-run "huggingface.co/owner/repo/blob/main/models"

Notes:
  - Requires: pip install huggingface_hub
  - Authentication (if needed) is taken from env HF_TOKEN or huggingface_hub.login()
"""

from __future__ import annotations

import argparse
import sys
import re
from urllib.parse import urlparse, unquote
from typing import Optional, Tuple, List

# huggingface_hub API
try:
    from huggingface_hub import hf_hub_download, snapshot_download
except Exception:
    hf_hub_download = None
    snapshot_download = None


# ----------------------------------------------------------------------
# Regex patterns
# ----------------------------------------------------------------------
RE_BLOB_RESOLVE = re.compile(
    r'^(?:https?://)?(?:www\.)?huggingface\.co/'
    r'(?P<repo>[^/]+/[^/]+)/(?:blob|resolve)/'
    r'(?P<rev>[^/]+)/(?P<path>.+)$'
)

RE_NO_PREFIX = re.compile(
    r'^(?P<repo>[^/]+/[^/]+)/(?:blob|resolve)/'
    r'(?P<rev>[^/]+)/(?P<path>.+)$'
)

RE_SIMPLE = re.compile(
    r'^(?P<repo>[^/]+/[^/]+)(?:/(?P<path>.+))?$'
)


# ----------------------------------------------------------------------
# Input parser
# ----------------------------------------------------------------------
def parse_input(s: str) -> Optional[Tuple[str, Optional[str], str]]:
    """
    Parse input and return (repo, revision, path)

    revision may be None (meaning default branch)
    """
    s = s.strip()
    s = unquote(s.split('?', 1)[0].split('#', 1)[0]).rstrip('/')

    # 1) Explicit Hugging Face URL (blob/resolve)
    m = RE_BLOB_RESOLVE.match(s)
    if m:
        return m.group('repo'), m.group('rev'), m.group('path')

    # 2) huggingface.co/... without scheme
    if s.startswith('huggingface.co/'):
        candidate = s[len('huggingface.co/'):].lstrip('/')
        m2 = RE_NO_PREFIX.match(candidate)
        if m2:
            return m2.group('repo'), m2.group('rev'), m2.group('path')

        m2 = RE_SIMPLE.match(candidate)
        if m2 and m2.group('path'):
            return m2.group('repo'), None, m2.group('path')

    # 3) Generic URL parse
    try:
        p = urlparse(s)
    except Exception:
        p = None

    if p and p.netloc and 'huggingface' in p.netloc:
        parts = p.path.lstrip('/').split('/')
        if len(parts) >= 5 and parts[2] in ('blob', 'resolve'):
            repo = f"{parts[0]}/{parts[1]}"
            rev = parts[3]
            path = '/'.join(parts[4:])
            return repo, rev, path
        elif len(parts) >= 3:
            repo = f"{parts[0]}/{parts[1]}"
            path = '/'.join(parts[2:])
            return repo, None, path

    # 4) Direct repo/path
    m3 = RE_NO_PREFIX.match(s)
    if m3:
        return m3.group('repo'), m3.group('rev'), m3.group('path')

    m4 = RE_SIMPLE.match(s)
    if m4 and m4.group('path'):
        return m4.group('repo'), None, m4.group('path')

    return None


# ----------------------------------------------------------------------
# Download logic
# ----------------------------------------------------------------------
def run_hf_download_api(
    repo: str,
    path: str,
    rev: Optional[str],
    local_dir: Optional[str],
    dry_run: bool,
) -> int:
    """
    Try single-file download first; if it fails, fall back to snapshot_download
    (directory or pattern).
    """
    if hf_hub_download is None or snapshot_download is None:
        print(
            "Error: huggingface_hub is not installed. "
            "Please run `pip install huggingface_hub`.",
            file=sys.stderr,
        )
        return 2

    if local_dir is None:
        local_dir = repo

    rev_disp = rev if rev else "default"
    print(f"> (api) download {repo}@{rev_disp} {path} -> local_dir={local_dir}")

    if dry_run:
        return 0

    # ---- Try as single file ----
    try:
        local_path = hf_hub_download(
            repo_id=repo,
            filename=path,
            revision=rev,
            local_dir=local_dir,
        )
        print(f"Downloaded file: {local_path}")
        return 0
    except Exception as e:
        print(
            f"hf_hub_download failed: {e}. "
            "Trying snapshot_download for directory/pattern...",
            file=sys.stderr,
        )

    # ---- Fallback: directory or glob ----
    allow_pattern = path.rstrip("/") + "/*"

    try:
        repo_local_dir = snapshot_download(
            repo_id=repo,
            revision=rev,
            local_dir=local_dir,
            allow_patterns=[allow_pattern],
        )
        print(f"Snapshot downloaded into: {repo_local_dir}")
        return 0
    except Exception as e:
        print(f"snapshot_download failed: {e}", file=sys.stderr)
        return 3


# ----------------------------------------------------------------------
# Main
# ----------------------------------------------------------------------
def main(argv: List[str]) -> int:
    parser = argparse.ArgumentParser(
        description="Download files or directories from Hugging Face Hub"
    )
    parser.add_argument(
        "inputs",
        nargs="+",
        help="Hugging Face URL or <namespace/repo>/path",
    )
    parser.add_argument(
        "--dry-run",
        action="store_true",
        help="Print actions without executing",
    )
    parser.add_argument(
        "--hf-cmd",
        default="hf",
        help="(ignored, kept for compatibility)",
    )
    parser.add_argument(
        "--flatten-localdir",
        action="store_true",
        help="Replace '/' with '-' in local directory name",
    )
    args = parser.parse_args(argv)

    any_failed = False

    for s in args.inputs:
        parsed = parse_input(s)
        if not parsed:
            print(f"Failed to parse input: {s}", file=sys.stderr)
            any_failed = True
            continue

        repo, rev, path = parsed
        if not path:
            print(f"No file path extracted for input: {s}", file=sys.stderr)
            any_failed = True
            continue

        local_dir = repo.replace("/", "-") if args.flatten_localdir else repo

        rc = run_hf_download_api(
            repo=repo,
            path=path,
            rev=rev,
            local_dir=local_dir,
            dry_run=args.dry_run,
        )
        if rc != 0:
            any_failed = True

    return 1 if any_failed else 0


if __name__ == "__main__":
    sys.exit(main(sys.argv[1:]))
```
