---
title: "OCG Configuration Tool for Aligning PDF Display and Print Output"
description: "The content of a PDF can sometimes differ between what you see on screen and what gets printed.  One..."
pubDatetime: 2026-06-12T08:28:40.352Z
updatedDate: 2026-06-12T08:33:02.911Z
---

The content of a PDF can sometimes differ between what you see on screen and what gets printed.

One common cause is **OCG (Optional Content Groups)**, which are layer definitions in a PDF that can be shown or hidden depending on viewing, printing, or export settings.

This tool is designed to help when OCG settings cause discrepancies between on-screen display and printed output by **making the print state closely match the current display state**.

## Purpose

The purpose of this tool is to synchronize the display, print, and export states of each layer in a PDF.

In many PDFs, layers that are visible on screen may be hidden during printing. As a result, content that appears correctly while viewing the PDF may disappear when printed, or content hidden on screen may unexpectedly appear in the printed output.

Using this tool, the current layer state of the PDF is used to configure the print and export settings so that they follow the same visibility state as the display configuration.

## Typical Use Cases

This tool is useful in situations such as:

* Printing a PDF exactly as it appears on screen
* When certain text, diagrams, annotations, or backgrounds disappear during printing
* When layer settings cause display and print results to differ
* Preparing OCG-enabled PDFs for more predictable output behavior

## Usage

The basic usage is to specify an input PDF and an output PDF.

```bash
python sync_pdf_ocg_print_state.py input.pdf output.pdf
```

This command examines the OCG configuration of `input.pdf`, applies the current display state to the print/view/export settings, and saves the result as `output.pdf`.

## Listing Layers

To see which OCG layers are present in a PDF, run:

```bash
python sync_pdf_ocg_print_state.py input.pdf --list-layers
```

This displays the layer numbers and layer names contained in the PDF.

It is useful when you want to verify whether a PDF contains OCG layers before processing it.

## Preview Changes Without Modifying the PDF

To see what changes would be applied without creating a new PDF, use `--dry-run`:

```bash
python sync_pdf_ocg_print_state.py input.pdf --dry-run
```

In this mode, no output PDF is generated.

The tool simply reports which layers would be treated as ON or OFF.

## Display Detailed Information

To view information about the original layer configuration during processing, add `-v` or `--verbose`:

```bash
python sync_pdf_ocg_print_state.py input.pdf output.pdf --verbose
```

This displays a summary of the effective BaseState and related layer settings used by the PDF.

## Notes

This tool is only effective for PDFs that contain OCG layers.

If a PDF does not contain any OCGs, there are no layers to process and the tool cannot make any changes.

It is also recommended to save the processed file as a new PDF rather than overwriting the original file. Keeping the original PDF allows you to revert if necessary.

## Summary

This is a simple utility for situations where OCG settings cause differences between PDF display and printed output.

By propagating the current display state to the print and export settings, it helps ensure that printed output more closely matches what you see on screen.

```python
#!/usr/bin/env python3
"""sync_pdf_ocg_print_state.py

Set OCG Usage state metadata for all Optional Content Groups.

Default behavior:
- Resolve each OCG effective BaseState from OCProperties / get_layer(-1).
- Write that state to:
    /Usage << /Print << /PrintState ... >> /View << /ViewState ... >> /Export << /ExportState ... >> >>
- Applies to all OCGs by default; layer selection options are intentionally removed.
"""

from __future__ import annotations

import argparse
from pathlib import Path
from typing import Optional

try:
    import pymupdf as fitz  # type: ignore
except ImportError:  # pragma: no cover
    import fitz  # type: ignore


def _normalize_state(raw: str | None) -> str:
    if not raw:
        return "/ON"
    s = raw.strip()
    return s if s.startswith("/") else f"/{s}"


def _parse_refs(raw: str | None) -> set[int]:
    if not raw:
        return set()
    out: set[int] = set()
    parts = raw.replace("\r", " ").replace("\n", " ").split()
    for i in range(0, len(parts) - 2, 3):
        if parts[i + 1] == "0" and parts[i + 2] == "R" and parts[i].isdigit():
            out.add(int(parts[i]))
    return out


def _get_catalog_xref(doc: fitz.Document) -> Optional[int]:
    try:
        c = doc.pdf_catalog()
        return int(c) if c else None
    except Exception:
        for xref in range(1, doc.xref_length()):
            try:
                obj = doc.xref_object(xref, compressed=False)
            except Exception:
                continue
            if "/Type /Catalog" in obj or "/Type/Catalog" in obj:
                return xref
    return None


def _get_basestate_map(
    doc: fitz.Document,
    ocg_xrefs: set[int],
) -> tuple[str, set[int], set[int]]:
    """Return (base_state, on_set, off_set) for effective BaseState evaluation."""
    base_state = "/ON"
    on_set: set[int] = set()
    off_set: set[int] = set()

    if hasattr(doc, "get_layer"):
        try:
            cfg = doc.get_layer(-1)
            if isinstance(cfg, dict):
                raw_base = cfg.get("basestate") or cfg.get("base_state") or cfg.get("BaseState")
                if raw_base:
                    base_state = _normalize_state(str(raw_base).strip("'\""))
                on_raw = cfg.get("on") or cfg.get("ON") or []
                off_raw = cfg.get("off") or cfg.get("OFF") or []
                on_set = {
                    int(v) for v in on_raw if isinstance(v, int) or (isinstance(v, str) and v.isdigit())
                }
                off_set = {
                    int(v) for v in off_raw if isinstance(v, int) or (isinstance(v, str) and v.isdigit())
                }
                return base_state, on_set & ocg_xrefs, off_set & ocg_xrefs
        except Exception:
            pass

    catalog = _get_catalog_xref(doc)
    if not catalog:
        return base_state, on_set, off_set

    try:
        t_base, v_base = doc.xref_get_key(catalog, "OCProperties/D/BaseState")
        if t_base != "null" and v_base:
            base_state = _normalize_state(v_base)
        t_on, v_on = doc.xref_get_key(catalog, "OCProperties/D/ON")
        t_off, v_off = doc.xref_get_key(catalog, "OCProperties/D/OFF")
        if t_on != "null":
            on_set = _parse_refs(v_on)
        if t_off != "null":
            off_set = _parse_refs(v_off)
    except Exception:
        pass

    return base_state, (on_set & ocg_xrefs), (off_set & ocg_xrefs)


def _is_ocg_on_by_basestate(
    xref: int,
    base_state: str,
    on_set: set[int],
    off_set: set[int],
) -> bool:
    if base_state == "/OFF":
        return xref in on_set
    return xref not in off_set


def _collect_ocgs(doc: fitz.Document) -> dict[int, str]:
    ocgs: dict[int, str] = {}

    if hasattr(doc, "get_ocgs"):
        try:
            raw = doc.get_ocgs()
            for xref, info in raw.items():
                if isinstance(info, dict):
                    ocgs[int(xref)] = str(info.get("name", f"(xref {int(xref)})"))
        except Exception:
            ocgs = {}

    if ocgs:
        return ocgs

    for xref in range(1, doc.xref_length()):
        try:
            obj = doc.xref_object(xref, compressed=False)
        except Exception:
            continue
        if "/Type /OCG" not in obj and "/Type/OCG" not in obj:
            continue
        marker = "/Name"
        pos = obj.find(marker)
        if pos < 0:
            ocgs[xref] = f"(xref {xref})"
            continue
        start = obj.find("(", pos)
        if start < 0:
            ocgs[xref] = f"(xref {xref})"
            continue
        end = obj.find(")", start + 1)
        ocgs[xref] = obj[start + 1 : end] if end > start else f"(xref {xref})"
    return ocgs


def _ensure_usage_dict(doc: fitz.Document, ocg_xref: int) -> int:
    usage_typ, usage_val = doc.xref_get_key(ocg_xref, "Usage")

    if usage_typ == "xref" and usage_val:
        return int(usage_val.split(" ")[0])

    if usage_typ in {"null", "none"}:
        doc.xref_set_key(ocg_xref, "Usage", "<<>>")
        return ocg_xref

    if usage_typ:
        return ocg_xref

    doc.xref_set_key(ocg_xref, "Usage", "<<>>")
    return ocg_xref


def _set_usage_states(
    doc: fitz.Document,
    ocg_xref: int,
    *,
    state: str,
) -> None:
    target = _ensure_usage_dict(doc, ocg_xref)
    doc.xref_set_key(target, "Print/PrintState", state)
    doc.xref_set_key(target, "View/ViewState", state)
    doc.xref_set_key(target, "Export/ExportState", state)


def _list_layers(ocgs: dict[int, str]) -> None:
    if not ocgs:
        print("No OCG layers found.")
        return
    print("xref\tname")
    for xref in sorted(ocgs):
        print(f"{xref}\t{ocgs[xref]}")


def build_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser(
        description=(
            "Apply OCG Usage state metadata to all layers. "
            "Default mode derives state from BaseState and applies Print/View/Export states."
        )
    )
    p.add_argument("input", help="Input PDF")
    p.add_argument("output", nargs="?", help="Output PDF")
    p.add_argument(
        "--list-layers",
        action="store_true",
        help="List OCG xref and name, then exit",
    )
    p.add_argument(
        "--dry-run",
        action="store_true",
        help="Do not write output; show planned state for each OCG",
    )
    p.add_argument(
        "-v",
        "--verbose",
        action="store_true",
        help="Show effective BaseState summary",
    )
    return p


def main(argv: list[str] | None = None) -> int:
    args = build_parser().parse_args(argv)

    if not args.output and not (args.list_layers or args.dry_run):
        raise SystemExit("output is required unless --list-layers or --dry-run is used")

    doc = fitz.open(Path(args.input))
    try:
        ocgs = _collect_ocgs(doc)

        if args.list_layers:
            _list_layers(ocgs)
            return 0

        if not ocgs:
            raise SystemExit("No OCG layers found.")

        base_state, base_on, base_off = _get_basestate_map(doc, set(ocgs.keys()))
        if args.verbose:
            print(f"BaseState: {base_state}, ON refs: {sorted(base_on)}, OFF refs: {sorted(base_off)}")

        if args.dry_run:
            for xref in sorted(ocgs.keys()):
                state = "/ON" if _is_ocg_on_by_basestate(xref, base_state, base_on, base_off) else "/OFF"
                print(f"Would update xref {xref} ({ocgs[xref]}) state = {state} for Print/View/Export")
            return 0

        for xref in sorted(ocgs.keys()):
            state = "/ON" if _is_ocg_on_by_basestate(xref, base_state, base_on, base_off) else "/OFF"
            _set_usage_states(doc, xref, state=state)

        doc.save(Path(args.output), garbage=4, deflate=True, clean=True, incremental=False)
        print(f"Wrote: {Path(args.output)}")
        return 0
    finally:
        doc.close()


if __name__ == "__main__":
    raise SystemExit(main())
```
