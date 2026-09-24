---
title: "A Simple Python Tool for Controlled PDF Text Extraction (PyPDF)"
description: "Extract PDF text with a Python CLI that supports font filtering, sentence line breaks, hyphenated word merging, and streaming output for further processing."
pubDatetime: 2026-01-19T08:48:19.621Z
---

This script is a compact, command-line Python program designed to extract text from PDF files in a controlled and predictable way. Built on top of the `pypdf` library, it focuses on reliability rather than visual layout, making it suitable for preprocessing documents before analysis or conversion.

At its core, the program reads a PDF page by page and collects text fragments directly from the content stream. Font-based filtering can be enabled to extract only text rendered with specific font names and sizes, but by default the filter is disabled so that all text is captured.

Key behaviors are implemented to improve readability of the output and reduce common PDF artifacts:

* Optional filtering by exact font name and font size with tolerance
* Automatic insertion of line breaks after periods
* Intelligent merging of hyphenated line endings
* Streaming output to standard output for easy piping
* Minimal configuration centralized at the top of the script

Overall, the script provides a practical balance between simplicity and control, making it useful for batch processing PDFs or integrating into larger text-processing workflows.

```python
#!/usr/bin/env python3
from __future__ import annotations

import math
import sys
from typing import Iterator, Optional, Tuple

from pypdf import PdfReader

# =========================
# Extraction conditions (adjust only here if needed)
# =========================
TARGET_FONTS = {
    ("Hoge", 12.555059999999997),
    ("Fuga", 12.945840000000032),
}
SIZE_TOL = 1e-6  # Tolerance for math.isclose

# As in the original code, extraction of all text (font filter disabled) is the default
ENABLE_FONT_FILTER = False


def _normalize_font_name(raw) -> Optional[str]:
    """
    Convert and normalize font information passed from pypdf into a string.
    Example: NameObject('/Hoge') -> 'Hoge'
    """
    if raw is None:
        return None
    s = str(raw)
    if s.startswith("/"):
        s = s[1:]
    return s or None


def is_target_text(font_name: Optional[str], font_size: Optional[float]) -> bool:
    """Determine whether a text fragment is a target for extraction (by font name and size)."""
    if not ENABLE_FONT_FILTER:
        return True

    if font_name is None or font_size is None:
        return False

    for f, sz in TARGET_FONTS:
        if font_name == f and math.isclose(font_size, sz, rel_tol=0.0, abs_tol=SIZE_TOL):
            return True
    return False


def extract_text_stream(fp) -> Iterator[str]:
    """
    - Extract only target text (optionally filtered by font name and size)
    - Replace '.' with '.\\n'
    - If a line ends with '-', merge it with the next line (remove the trailing '-')
    """
    reader = PdfReader(fp)

    carry = ""  # Buffer for joining lines when a line ends with a hyphen

    for page in reader.pages:
        chunks: list[str] = []

        def visitor_text(
            text: str,
            cm,  # current transformation matrix
            tm,  # text matrix
            font_dict,
            font_size: float,
        ):
            # Guard because text may be empty
            if not text:
                return

            # font_dict is often a dict-like object (some PDFs may not provide it)
            base_font = None
            try:
                if font_dict:
                    base_font = font_dict.get("/BaseFont")
            except Exception:
                base_font = None

            font_name = _normalize_font_name(base_font)
            size = float(font_size) if font_size is not None else None

            if is_target_text(font_name, size):
                chunks.append(text)

        # Using visitor_text allows collecting text fragments
        # in the order of the content stream
        # extraction_mode is not specified because it may be unsupported
        # depending on the pypdf version
        page.extract_text(visitor_text=visitor_text)

        s = "".join(chunks)
        if not s:
            continue

        s = s.replace(".", ".\n")

        for line in s.splitlines(keepends=False):
            if carry:
                line = carry + line
                carry = ""

            if line.endswith("-"):
                carry = line[:-1]
                continue

            yield line

        # As in the original code, we do not flush at block boundaries;
        # carry is preserved (you can flush here if needed)
        # if carry:
        #     yield carry
        #     carry = ""

    if carry:
        yield carry


def main(pdf_path: str) -> None:
    with open(pdf_path, "rb") as f:
        for chunk in extract_text_stream(f):
            sys.stdout.buffer.write(chunk.encode() + b"\n")


if __name__ == "__main__":
    path = sys.argv[1]
    main(path)
```
