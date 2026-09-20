---
title: "PDF Page Combining Tool"
description: "Overview   This tool combines multiple pages from a PDF into a grid layout on each output..."
pubDatetime: 2026-06-12T07:15:53.337Z
---

## Overview

This tool combines multiple pages from a PDF into a grid layout on each output page. It is useful when you want to place several PDF pages together, such as arranging four pages into a 2 × 2 layout.

The tool can be used to reduce the number of pages, prepare handouts, create overview sheets, or make compact versions of PDF documents.

## Main Purpose

The main purpose of this tool is to create an **n-up PDF**.

“N-up” means placing multiple pages onto one sheet. For example, a 2 × 2 layout places four original PDF pages on one output page.

The tool can:

* Select all pages or only specific pages from the input PDF
* Arrange pages in rows and columns
* Create larger output pages without shrinking the original pages
* Shrink pages to fit into the original page size
* Add margins around the output page
* Add gaps between the arranged pages

## How It Works

The tool reads an input PDF, selects the requested pages, and places them into a grid. Each group of pages becomes one output page.

For example, if the layout is set to 2 columns and 2 rows, the tool places up to four source pages on each output page.

Pages are placed from left to right and from top to bottom.

## Selecting Pages

You can choose which pages to include by using a page range.

If no page range is given, the tool uses all pages in the PDF.

Examples:

| Page Range | Meaning                          |
| ---------- | -------------------------------- |
| `1-4`      | Use pages 1 through 4            |
| `1,3,5-8`  | Use pages 1, 3, and 5 through 8  |
| `2-`       | Use page 2 through the last page |
| `-10`      | Use page 1 through page 10       |
| empty      | Use all pages                    |

Page numbers are written in the normal 1-based style, so page `1` means the first page of the PDF.

## Basic Usage

Run the tool from the command line by giving an input PDF and an output PDF:

```bash
python nup_pdf.py input.pdf output.pdf
```

By default, this creates a 2 × 2 layout using all pages.

## Choosing Rows and Columns

You can change the layout with `--cols` and `--rows`.

For example, this creates a 3 × 2 layout:

```bash
python nup_pdf.py input.pdf output.pdf --cols 3 --rows 2
```

This places up to six original pages on each output page.

## Using a Page Range

To use only selected pages, add `--range`.

```bash
python nup_pdf.py input.pdf output.pdf --range "1-4"
```

This uses only pages 1 through 4.

You can also combine separate pages and ranges:

```bash
python nup_pdf.py input.pdf output.pdf --range "1,3,5-8"
```

## Keeping the Original Page Size

By default, the tool does not shrink the source pages. Instead, it creates a larger output page large enough to hold the full grid.

For example, if four A4 pages are arranged in a 2 × 2 layout, the output page will be roughly A2-sized.

This is useful when you want to preserve the original page size and avoid reducing text or image quality.

## Scaling Pages to Fit

Use `--scale-to-fit` when you want the output page to stay the same size as the first page of the input PDF.

```bash
python nup_pdf.py input.pdf output.pdf --scale-to-fit
```

In this mode, each source page is reduced so that it fits inside its grid cell.

This is closer to ordinary printer-style n-up output, where multiple pages are placed on one sheet.

## Adding Margins and Gaps

You can add an outer margin with `--margin` and spacing between grid cells with `--gap`.

```bash
python nup_pdf.py input.pdf output.pdf --margin 20 --gap 10
```

Both values are measured in PDF points.

One point is 1/72 inch.

Margins are added around the outside of the output page. Gaps are added between the placed pages.

## Example Commands

### Create a 2 × 2 PDF using all pages

```bash
python nup_pdf.py input.pdf output.pdf
```

### Create a 2 × 2 PDF using only pages 1 to 8

```bash
python nup_pdf.py input.pdf output.pdf --range "1-8"
```

### Create a 3 × 2 layout

```bash
python nup_pdf.py input.pdf output.pdf --cols 3 --rows 2
```

### Create a printer-style compact version

```bash
python nup_pdf.py input.pdf output.pdf --scale-to-fit
```

### Add margin and spacing

```bash
python nup_pdf.py input.pdf output.pdf --margin 24 --gap 12
```

## When to Use This Tool

This tool is helpful when you need to:

* Make handouts from slide PDFs
* Combine several document pages into one overview page
* Reduce the number of printed sheets
* Create compact review materials
* Arrange selected PDF pages into a clean grid

## Summary

This PDF page combining tool creates a new PDF by arranging selected pages from an existing PDF into a grid. It supports flexible page selection, custom rows and columns, optional scaling, margins, and gaps.

Use the default mode when you want to preserve original page sizes. Use `--scale-to-fit` when you want several pages to fit onto a page of the original size.

```python
from __future__ import annotations

from pathlib import Path

from pypdf import PdfReader, PdfWriter, PageObject, Transformation


def parse_page_range(page_range: str | None, total_pages: int) -> list[int]:
    """
    Convert a 1-based page range string to a list of 0-based page indices.

    If page_range is None or empty, all pages are selected.

    Examples
    --------
    "1-4"     -> [0, 1, 2, 3]
    "1,3,5-7" -> [0, 2, 4, 5, 6]
    "2-"      -> from page 2 to the last page
    "-5"      -> from page 1 to page 5
    "" or None -> all pages
    """
    if page_range is None or not page_range.strip():
        return list(range(total_pages))

    pages: list[int] = []

    for part in page_range.split(","):
        part = part.strip()
        if not part:
            continue

        if "-" in part:
            start_s, end_s = part.split("-", 1)

            start = int(start_s) if start_s else 1
            end = int(end_s) if end_s else total_pages

            if start < 1 or end > total_pages or start > end:
                raise ValueError(f"Invalid page range: {part}")

            pages.extend(range(start - 1, end))
        else:
            p = int(part)
            if p < 1 or p > total_pages:
                raise ValueError(f"Page does not exist: {p}")
            pages.append(p - 1)

    if not pages:
        return list(range(total_pages))

    return pages


def nup_pdf(
    input_pdf: str | Path,
    output_pdf: str | Path,
    page_range: str | None = None,
    cols: int = 2,
    rows: int = 2,
    *,
    scale_to_fit: bool = False,
    margin: float = 0,
    gap: float = 0,
) -> None:
    """
    Combine selected PDF pages into a cols x rows grid on each output page.

    Parameters
    ----------
    input_pdf:
        Input PDF file.
    output_pdf:
        Output PDF file.
    page_range:
        Page range to use, expressed with 1-based page numbers.
        If omitted, all pages are used.
        Examples: "1-4", "1,3,5-8", "2-", "-10"
    cols:
        Number of pages in the horizontal direction. Use 2 for a 2x2 layout.
    rows:
        Number of pages in the vertical direction. Use 2 for a 2x2 layout.
    scale_to_fit:
        False:
            Preserve the original page size and create a larger output page.
            For example, a 2x2 layout of A4 pages is roughly A2-sized.
        True:
            Keep the output page size equal to the first input page and shrink
            each source page into its grid cell. This is closer to ordinary
            n-up printing behavior.
    margin:
        Outer margin of the output page, in PDF points.
        1 point = 1/72 inch.
    gap:
        Gap between cells, in PDF points.
    """
    if cols <= 0 or rows <= 0:
        raise ValueError("cols and rows must be 1 or greater.")

    reader = PdfReader(str(input_pdf))
    writer = PdfWriter()

    selected_indices = parse_page_range(page_range, len(reader.pages))
    if not selected_indices:
        raise ValueError("No pages were selected.")

    pages_per_sheet = cols * rows

    # Reference page size.
    first_page = reader.pages[selected_indices[0]]
    base_width = float(first_page.mediabox.width)
    base_height = float(first_page.mediabox.height)

    if scale_to_fit:
        # Keep the output size equal to one original PDF page.
        out_width = base_width
        out_height = base_height

        cell_width = (out_width - 2 * margin - gap * (cols - 1)) / cols
        cell_height = (out_height - 2 * margin - gap * (rows - 1)) / rows
    else:
        # Do not shrink pages; create a larger page for the full grid.
        cell_width = base_width
        cell_height = base_height

        out_width = cols * cell_width + gap * (cols - 1) + 2 * margin
        out_height = rows * cell_height + gap * (rows - 1) + 2 * margin

    for group_start in range(0, len(selected_indices), pages_per_sheet):
        group = selected_indices[group_start : group_start + pages_per_sheet]

        output_page = PageObject.create_blank_page(
            width=out_width,
            height=out_height,
        )

        for slot, page_index in enumerate(group):
            src_page = reader.pages[page_index]

            src_width = float(src_page.mediabox.width)
            src_height = float(src_page.mediabox.height)

            col = slot % cols
            row = slot // cols

            # The PDF coordinate system starts at the lower-left corner.
            # Flip the y-position calculation so row=0 is placed on the top row.
            x0 = margin + col * (cell_width + gap)
            y0 = margin + (rows - 1 - row) * (cell_height + gap)

            if scale_to_fit:
                scale = min(cell_width / src_width, cell_height / src_height)
            else:
                scale = 1.0

            placed_width = src_width * scale
            placed_height = src_height * scale

            # Center the source page within the cell.
            tx = x0 + (cell_width - placed_width) / 2
            ty = y0 + (cell_height - placed_height) / 2

            # Also support PDFs whose mediabox lower-left corner is not (0, 0).
            src_left = float(src_page.mediabox.left)
            src_bottom = float(src_page.mediabox.bottom)

            transform = (
                Transformation()
                .translate(tx=-src_left, ty=-src_bottom)
                .scale(scale, scale)
                .translate(tx=tx, ty=ty)
            )

            output_page.merge_transformed_page(
                src_page,
                transform,
                over=True,
            )

        writer.add_page(output_page)

    with open(output_pdf, "wb") as f:
        writer.write(f)


import argparse


def main() -> None:
    parser = argparse.ArgumentParser(
        description="Combine PDF pages into an n-up grid layout."
    )
    parser.add_argument("input_pdf")
    parser.add_argument("output_pdf")
    parser.add_argument(
        "--range",
        dest="page_range",
        default=None,
        help="1-based page range to use, such as '1-4', '1,3,5-8', '2-', or '-10'. If omitted, all pages are used.",
    )
    parser.add_argument("--cols", type=int, default=2)
    parser.add_argument("--rows", type=int, default=2)
    parser.add_argument("--scale-to-fit", action="store_true")
    parser.add_argument("--margin", type=float, default=0)
    parser.add_argument("--gap", type=float, default=0)

    args = parser.parse_args()

    nup_pdf(
        input_pdf=args.input_pdf,
        output_pdf=args.output_pdf,
        page_range=args.page_range,
        cols=args.cols,
        rows=args.rows,
        scale_to_fit=args.scale_to_fit,
        margin=args.margin,
        gap=args.gap,
    )


if __name__ == "__main__":
    main()
```
