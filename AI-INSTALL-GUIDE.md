# AI Install Guide: solo-pdf2md

**Audience: an AI coding agent.** This document is self-contained. It holds the
complete installation procedure *and* the complete source code (Appendix A), so
you can set the tool up without fetching anything from the network.

Copy this entire file into your context, then follow the steps in order.

---

## 1. Objective

Install and verify `solo-pdf2md`: a single-file Python CLI that converts a
research-paper PDF into Markdown and then removes the reference list, so that
reading the paper costs fewer context tokens. Everything runs locally.

**Order of operations matters and is handled internally by the tool:** the PDF is
first converted to Markdown (via `pymupdf4llm`), and only then is the reference
list located and cut in the *Markdown text*. A PDF is a binary stream, so a
regex cannot find `References` inside it directly. Do not attempt to reverse
this order.

## 2. Deliverables

When you are finished, all of the following must be true:

1. A directory containing `solo-pdf2md.py` (byte-identical to Appendix A).
2. A virtual environment with `pymupdf4llm` installed and importable.
3. A passing self-test (Step 4) whose output you have actually read.
4. A short report to the user: where the tool was installed, the exact commands
   used, the self-test result, and any warnings.

## 3. Prerequisites

- Python 3.10 or newer. Check with `python3 --version`.
- Roughly 200 MB of free disk space (the layout model ships inside the
  `pymupdf4llm` wheel and its dependencies).
- Network access for `pip install` (one time only). If there is no network
  access, stop and tell the user; do not fake the installation.

Optional: [Tesseract](https://github.com/tesseract-ocr/tesseract) for OCR of
scanned PDFs (`brew install tesseract` on macOS, `apt install tesseract-ocr` on
Debian/Ubuntu). The tool works without it for PDFs that have a text layer.

## 4. Installation procedure

### Step 1 - Create the project directory

```bash
mkdir -p "$HOME/solo-pdf2md"
cd "$HOME/solo-pdf2md"
```

Use a different path if the user specified one.

### Step 2 - Create a virtual environment and install the dependency

```bash
python3 -m venv .venv
.venv/bin/pip install --upgrade pip
.venv/bin/pip install "pymupdf4llm>=1.28.2"
```

On Windows the interpreter path is `.venv\Scripts\python.exe` instead of
`.venv/bin/python`.

Confirm it imports before continuing:

```bash
.venv/bin/python -c "import pymupdf4llm, pymupdf; print('pymupdf4llm OK', pymupdf.version)"
```

If this fails, resolve it before moving on. Do not proceed with a broken
environment.

### Step 3 - Write the source file

Create `solo-pdf2md.py` in the project directory with **exactly** the contents of
Appendix A. Do not reformat, retype, or "improve" the code: several regexes are
deliberately shaped to handle real-world publisher formatting, and small edits
will silently break them.

Then confirm the file parses:

```bash
.venv/bin/python -c "import ast; ast.parse(open('solo-pdf2md.py').read()); print('syntax OK')"
```

### Step 4 - Verify (do not skip)

Two tests. The first is deterministic and must pass; the second needs a PDF.

**Test 4a - self-test of the cutting logic.** The reference-detection logic is a
pure text function, so it can be tested without any PDF. Write the file in
Appendix B to `selftest.py` and run it:

```bash
.venv/bin/python selftest.py
```

Expected output:

```
PASS: heading cut
PASS: entry-run cut (no heading)
PASS: scrambled layout is refused
PASS: no references leaves body untouched
ALL TESTS PASSED
```

If any test does not pass, your copy of `solo-pdf2md.py` differs from Appendix A.
Rewrite the file from Appendix A and retry.

**Test 4b - end-to-end on a real PDF (optional).** If the user has a PDF
available, run:

```bash
.venv/bin/python solo-pdf2md.py /path/to/paper.pdf -o out/
```

Expect a line of the form:

```
[1/1] paper.pdf    13804 -> 11676 tok  cut(heading+tail-kept)
```

and two files in `out/`: `paper.md` and `paper.refs.md`. A `method` value of
`not-cut(...)` or `skipped-scrambled-layout` is **not** a crash: it is a deliberate
safety outcome. Report it as-is rather than trying to force a cut.

### Step 5 - Report

Summarise for the user: install path, the version of `pymupdf4llm`, the
self-test result, and - if Test 4b ran - the token figures printed by the tool.

## 5. Using the tool

```bash
# One PDF; output goes next to the input file
.venv/bin/python solo-pdf2md.py paper.pdf

# Explicit output directory
.venv/bin/python solo-pdf2md.py paper.pdf -o out/

# A whole directory, recursively
.venv/bin/python solo-pdf2md.py /path/to/pdfs -o out/

# Keep the reference list (it is cut by default)
.venv/bin/python solo-pdf2md.py paper.pdf --keep-refs

# Selected pages / forced OCR
.venv/bin/python solo-pdf2md.py paper.pdf --pages 1-5,8
.venv/bin/python solo-pdf2md.py scan.pdf --ocr-lang eng --force-ocr
```

| Flag | Meaning |
|---|---|
| `-o, --out` | Output directory. Default: the input PDF's own directory. |
| `--pages` | Page selection, e.g. `"1-5,8"`. |
| `--ocr-lang` | Tesseract language; default `eng`. |
| `--force-ocr` | OCR every page (scanned PDFs, or a broken text layer). |
| `--keep-refs` | Do not cut the reference list. |
| `--no-refs-file` | Do not write the separate `.refs.md` file. |

## 6. Output format

For an input `paper.pdf` the tool writes:

| File | Contents |
|---|---|
| `paper.md` | The body, with the reference list removed. |
| `paper.refs.md` | The reference list on its own, so citations can still be checked. |
| `_solo_index.json` | A manifest: per-file token counts, method used, warnings. |

The `method` field in the manifest is one of:

| `method` | Meaning |
|---|---|
| `heading` | Cut at a `References` / `Bibliography` heading. |
| `entry-run` | No heading found; cut at a detected run of numbered entries. |
| `heading+tail-kept` / `entry-run+tail-kept` | Content after the reference list (Methods, figure legends) was preserved. |
| `kept` | `--keep-refs` was used. |
| `skipped-scrambled-layout` | A safety guard tripped; nothing was cut. |
| *(empty string)* | No reference list was found. |

## 7. Troubleshooting

| Symptom | Cause and action |
|---|---|
| `Missing dependency: pymupdf4llm` | The venv is not active or the install failed. Re-run Step 2 and the import check. |
| `extraction produced no text (the PDF may be corrupt...)` | The PDF's xref table is unreadable. Report it; a re-download of the PDF is usually required. Do not fabricate content. |
| Garbled characters in the output | The PDF uses a broken CID font (common in older CJK typesetting). Re-run with `--force-ocr --ocr-lang eng` (or the appropriate language). |
| `method` is `skipped-scrambled-layout` | Two-column layout with broken reading order. This is intentional: the tool refuses to risk deleting body text. Nothing is lost. |
| Nonsense token counts | The counts are estimates (`~4 chars/token` for Latin text, `~1.5` for CJK), not a real tokenizer. They are for relative comparison only. |

## 8. Constraints to respect

- **Do not modify the algorithm** to make a test pass. The three safety guards
  exist because a naive positional cut was measured to delete Methods sections
  and figure legends from real papers.
- **Do not vendor or copy code from `pymupdf4llm`** into the project. It is used
  strictly through its public API (`pymupdf4llm.to_markdown`, `pymupdf.open`).
- **License:** this project is AGPL-3.0-or-later. `pymupdf4llm` is dual-licensed
  (AGPL-3.0 or an Artifex commercial license) and is used here under AGPL-3.0.
  Do not relicense the project, and do not remove the SPDX header from the source
  file. If the user asks about closed-source or SaaS use, tell them they need a
  commercial license from Artifex.
- **Report failures honestly.** Never invent output, token figures, or a passing
  test result.

---

## Appendix A - Full source: `solo-pdf2md.py`

Write this file verbatim. The four-backtick fence below is deliberate; the code
itself contains no such fence.

````python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
# SPDX-License-Identifier: AGPL-3.0-or-later
# Copyright (C) 2026 solo-pdf2md contributors

"""
solo-pdf2md.py -- self-contained single-file tool: PDF -> Markdown -> cut references
====================================================================================

Design goals:
  * Self-contained: the pymupdf4llm conversion call lives INSIDE this file;
    it does not depend on any sibling script.
  * Fixed behaviour: any PDF in, always "convert to Markdown first, then cut the
    reference list" -- no extra flags required.
  * Minimal output: just <name>.md (body) + <name>.refs.md (references, kept for
    lookups). No section index, no process pool; the logic stays readable.

Order of operations (important):
  pymupdf4llm conversion  ->  run regexes over the Markdown TEXT to cut references
  Not the other way round: a PDF is a binary stream, so a regex cannot match
  "References" directly inside the PDF.

Cutting strategy (three detection tiers + three safety guards):
  Detection: 1. Heading (References / Bibliography / ...)
             2. Numbered entry run (1. / [1] / with list bullets / with PDF line numbers)
             3. Backward extension (when a list is broken by captions or footers,
                pull in the earlier half too)
  Safety:    1. Cut only the reference region; keep Methods / figures / appendices
                that follow it.
             2. If the tail we were about to keep is itself reference-heavy,
                cut that too.
             3. On scrambled two-column layouts (body headings leaking into the
                region to be deleted), abandon the cut entirely -- better to
                spend extra tokens than to delete body text.

Usage:
  # simplest: one PDF, output next to the input
  python solo-pdf2md.py paper.pdf

  # choose an output directory
  python solo-pdf2md.py paper.pdf -o out/

  # a whole directory (recursive)
  python solo-pdf2md.py /path/to/PDFs -o out/

  # keep the references (they are cut by default)
  python solo-pdf2md.py paper.pdf --keep-refs

  # selected pages / forced OCR
  python solo-pdf2md.py paper.pdf --pages 1-5,8
  python solo-pdf2md.py scan.pdf --ocr-lang eng --force-ocr

Dependency:
  pymupdf4llm (dual-licensed: AGPL-3.0 or an Artifex commercial license).
  If it is missing, this script prints install instructions instead of failing
  silently.
"""

from __future__ import annotations

import argparse
import json
import re
import sys
import warnings
from pathlib import Path

warnings.filterwarnings("ignore")

# ---------------------------------------------------------------------------
# 1) Reference-detection rules
# ---------------------------------------------------------------------------

# Tier 1: heading lines (tolerating Markdown '#', bold '*', leading numbering,
# and a trailing colon or period).
# Note 1: the colon can sit INSIDE the bold markers ("**References and Notes:**"),
#         so '*' and punctuation must be allowed to interleave freely -- the
#         order cannot be hard-coded.
# Note 2: the leading character class must be [ \t*#] and NOT \s -- otherwise it
#         swallows the newline of the previous line, the match starts on the line
#         BEFORE the heading, and the cut point shifts up by one line.
REF_HEADING = re.compile(
    r"(?im)^[ \t*#]*"
    r"(?:\d+\.?\s*)?"
    r"(REFERENCES AND NOTES|LITERATURE CITED|REFERENCES CITED|WORKS CITED|"
    r"REFERENCES?|BIBLIOGRAPHY)"
    r"[\s*:.]*$"
)

# Tiers 2 and 3: reference entry lines.
# Real typesetting produces all sorts of prefixes, so this pattern tolerates:
#   plain          "1. Author ..."           "[1] Author ..."
#   with bullet    "- 1. Author ..."         (pymupdf4llm renders entries as a list)
#   with PDF line  "- 476 1. Author ..."     (Nature-style body text is line-numbered)
ENTRY = re.compile(
    r"^\s*"
    r"(?:[-*+]\s+)?"              # optional: Markdown list bullet
    r"(?:\d{1,5}\s+)?"            # optional: PDF line number
    r"(?:\[(\d{1,4})\]|(\d{1,4})[.)])\s+"   # reference number: [n] / n. / n)
    r"\S"
)
# Citation-flavoured features that should appear inside an entry: year, volume,
# page range, "et al", DOI.
CITE_FEATURE = re.compile(
    r"\((?:19|20)\d{2}[a-z]?\)"          # (2020) / (2020a)
    r"|\b(?:19|20)\d{2}\b"               # a bare year
    r"|\bet al\b"
    r"|\bdoi\b|https?://"
    r"|;\s*\d+\s*[:(]"                   # volume/issue forms such as ";12("
    r"|\b\d+\s*[-\u2013]\s*\d+\b"        # page ranges
)

MIN_ENTRY_CHARS = 55     # real entries are long; short lines are usually prose lists
MIN_RUN = 8              # how many consecutive entries make a reference list
NOISE_GAP = 3            # noise lines tolerated inside a run (running heads, wraps)
WRAP_GAP = 3             # wrapped continuation lines tolerated inside an entry
MIN_TAIL_CHARS = 400     # minimum remaining text before a post-reference tail counts


def _line_offsets(lines: list[str]) -> list[int]:
    """Character offset at the start of each line, for line<->offset conversion."""
    offs, acc = [], 0
    for ln in lines:
        offs.append(acc)
        acc += len(ln) + 1          # +1 restores the newline dropped by splitlines
    return offs


def _find_entry_run(lines: list[str]) -> int | None:
    """
    Find the LONGEST run of consecutive reference entries in the document.
    A few noise lines inside a run are tolerated so that running heads or
    wrapped lines do not split the whole list in two.
    """
    runs: list[tuple[int, int, int]] = []   # (start, last, count)
    start = last = None
    count = 0

    for i, raw in enumerate(lines):
        s = raw.strip()
        if not s:
            continue
        ok = (ENTRY.match(s)
              and len(s) >= MIN_ENTRY_CHARS
              and CITE_FEATURE.search(s))
        if ok:
            if start is None:
                start = i
            last, count = i, count + 1
            continue

        # Not an entry: accept it as noise as long as it is close to the last one.
        if last is not None and i - last <= NOISE_GAP:
            continue
        if start is not None and count >= MIN_RUN:
            runs.append((start, last, count))
        start = last = None
        count = 0

    if start is not None and count >= MIN_RUN:
        runs.append((start, last, count))
    if not runs:
        return None
    return max(runs, key=lambda r: r[2])[0]      # the run with the most entries


# Structural markers: section heading / table / bold sub-heading -- used to find
# where the reference region ends.
# The bold branch must require "a whole line starting with a letter", otherwise it
# matches bold fragments INSIDE entries (e.g. "... **312** , 927-930 (2006).")
# and truncates the reference region in the middle.
STRUCT_MARK = re.compile(
    r"^(?:#{1,6}\s|\||\*\*[A-Za-z][^*]{1,60}\*\*\s*:?\s*$)"
)
BOLD_HEADING = re.compile(r"^\*\*[A-Za-z][^*]{1,60}\*\*\s*:?\s*$")
MIN_ENTRIES_BEFORE_MARK = 2   # entries needed before a marker counts as post-reference content

# Body / back-matter headings. If one of these appears INSIDE the region about to
# be deleted, the two-column layout is interleaved (broken reading order) and a
# positional cut would delete real body text -> abandon the cut.
BODY_WORDS = re.compile(
    r"(?i)^(concluding|conclusion|discussion|results?|introduction|background|"
    r"methods?|materials?|funding|acknowledg|author contributions?|"
    r"competing interests?|conflict of interest|data availability|"
    r"supplement|appendix|abstract|tables?|figures?|editorial summary|"
    r"graphical abstract|highlights|limitations|future|perspectives)"
)
HEADING_LINE = re.compile(r"^#{1,6}\s+\S")


def _heading_text(line: str) -> str | None:
    """Clean a heading line down to plain text (strip #, **/_/`, and HTML tags)."""
    s = line.strip()
    if not HEADING_LINE.match(s):
        return None
    s = re.sub(r"<[^>]{1,40}>", "", s)        # <mark> / <u> / <br> and friends
    s = re.sub(r"^#{1,6}\s*", "", s)
    s = re.sub(r"[*_`>#]", "", s)
    return s.strip()


def _region_is_scrambled(lines: list[str], start: int, end: int) -> bool:
    """
    Check whether body or back-matter headings leak into the region to be cut.
    In normal typesetting a reference region contains entries only; once a body
    heading shows up inside it, the PDF is two-column with broken reading order
    and a positional cut would take body text with it.
    """
    for i in range(start, min(end, len(lines))):
        txt = _heading_text(lines[i])
        if txt and BODY_WORDS.match(txt):
            return True
    return False


def _ref_block_end(lines: list[str], start: int) -> int | None:
    """
    Find where the reference region ends (returns that line index, exclusive).

    Strategy: prefer the NEXT STRUCTURAL MARKER (section heading / table / bold
    sub-heading). This is more robust than tracking entry numbers one by one --
    many publishers (bioRxiv, for instance) interleave page furniture ("10",
    copyright lines, running heads) between entries, creating gaps wider than the
    tolerance, which makes entry tracking end the region far too early.

    When no structural marker is found (the reference list runs to the end of the
    document), fall back to entry tracking.
    """
    # --- 1) structural-marker pass ---
    entries_seen = 0
    for i in range(start + 1, len(lines)):
        s = lines[i].strip()
        if not s:
            continue
        if ENTRY.match(s) and len(s) >= 20:
            entries_seen += 1
            continue
        if STRUCT_MARK.match(s):
            if entries_seen >= MIN_ENTRIES_BEFORE_MARK:
                return i
            if i - start > 12:            # no entries showing up: not a reference region
                return None

    # --- 2) entry-tracking fallback ---
    last_entry = None
    seen = False
    for i in range(start, len(lines)):
        s = lines[i].strip()
        if not s:
            continue
        if ENTRY.match(s) and len(s) >= 20:
            last_entry, seen = i, True
            continue
        if not seen:
            if i - start > 12:
                return None
            continue
        if i - last_entry <= WRAP_GAP:
            continue
        break
    return last_entry + 1 if last_entry is not None else None


def _find_resume(lines: list[str], end: int) -> int | None:
    """
    After the reference region, find where body text or an appendix resumes:
    a Markdown heading, a table, or a bold sub-heading means the region is over.
    """
    for i in range(end, min(end + 8, len(lines))):
        s = lines[i].strip()
        if not s:
            continue
        if re.match(r"^#{1,6}\s", s):                 # "## Figure 1" / "#### Funding:"
            return i
        if s.startswith("|"):                          # supplementary table
            return i
        if BOLD_HEADING.match(s):                      # "**Funding:**"
            return i
    return None


def _line_of_offset(offs: list[int], cand: int) -> int:
    """Character offset -> line index."""
    lo, hi = 0, len(offs) - 1
    while lo < hi:
        mid = (lo + hi + 1) // 2
        if offs[mid] <= cand:
            lo = mid
        else:
            hi = mid - 1
    return lo


def _entries_after(lines: list[str], start: int, window: int = 140) -> int:
    """Count entries within `window` lines after `start`, to test if a heading is real."""
    n = 0
    for i in range(start + 1, min(start + window, len(lines))):
        s = lines[i].strip()
        if ENTRY.match(s) and len(s) >= 20:
            n += 1
    return n


def _extend_back(lines: list[str], start: int,
                 max_gap: int = 6, max_back: int = 400) -> int:
    """
    Extend the cut point backwards (towards the start of the file).

    Reference lists are often interrupted by figure captions or footers, so
    "find the longest consecutive entry run" may only locate the second half
    (starting at entry 27, say) while entries 1-26 are left in the body.
    This walks backwards while lines are still entries, or the gap stays within
    max_gap lines.
    """
    i = start - 1
    gap = 0
    best = start
    while i >= 0 and start - i <= max_back:
        s = lines[i].strip()
        if not s:
            i -= 1
            continue
        if ENTRY.match(s) and len(s) >= 20:
            best = i
            gap = 0
            i -= 1
            continue
        gap += 1
        if gap > max_gap:
            break
        i -= 1
    return best


def _tail_looks_like_refs(lines: list[str], resume: int, window: int = 60) -> bool:
    """
    Decide whether the tail we are about to KEEP is itself still a reference list.

    Reference lists get interrupted by captions and footers, so _find_resume may
    stop in the middle of the list, keeping its second half as if it were body
    text. This detects that by entry density and rejects it.
    """
    ent = tot = 0
    for i in range(resume, min(resume + window, len(lines))):
        s = lines[i].strip()
        if not s:
            continue
        tot += 1
        if ENTRY.match(s) and len(s) >= 20:
            ent += 1
    return tot >= 5 and ent >= 5 and ent / tot >= 0.5


def cut_references(md: str) -> tuple[str, list[str], str]:
    """
    Cut the reference list.
    Returns (body, reference_lines, method); when nothing matched, the body is
    returned unchanged and method is an empty string.

    Detection priority: heading > entry run. Neither blindly takes the last match,
    because the word "References" and numbered lists are both common in body text.
    """
    lines = md.splitlines()
    offs = _line_offsets(lines)
    cut_line: int | None = None
    how = ""

    # --- Tier 1: heading ---
    # Do not blindly take the LAST match: "References" can appear several times in
    # body text or appendices, and some candidates are a bare heading with no list
    # under them. Prefer the candidate followed by the most entries.
    heads = list(REF_HEADING.finditer(md))
    best_line = None
    best_count = -1
    for h in heads:
        if h.start() < len(md) * 0.15:       # guard: cutting that early means a misfire
            continue
        li = _line_of_offset(offs, h.start())
        cnt = _entries_after(lines, li)
        if cnt > best_count:                 # ties go to the earlier candidate
            best_line, best_count = li, cnt
    if best_line is not None and best_count >= 1:
        cut_line, how = best_line, "heading"
    elif heads:
        # No candidate is followed by entries: fall back to the last match
        # (a heading-only reference region).
        cand = heads[-1].start()
        if cand >= len(md) * 0.15:
            cut_line, how = _line_of_offset(offs, cand), "heading"

    # --- Tiers 2 and 3: entry run (fallback) ---
    if cut_line is None:
        idx = _find_entry_run(lines)
        if idx is not None:
            # Absorb a blank line before the run so it is not left dangling.
            while idx > 0 and not lines[idx - 1].strip():
                idx -= 1
            if offs[idx] >= len(md) * 0.15:
                cut_line, how = idx, "entry-run"

    if cut_line is None:
        return md, [], ""

    # When the list is broken by captions or footers, the lookup above may have
    # found only the second half; walk backwards to include the first half too.
    if how == "entry-run":
        cut_line = _extend_back(lines, cut_line)

    # --- Delimit the reference region, and decide whether anything follows it ---
    end = _ref_block_end(lines, cut_line)
    if end is None or end <= cut_line:
        end = len(lines)                     # cannot delimit entries: treat as end-of-file

    resume = _find_resume(lines, end) if end < len(lines) else None
    if resume is not None:
        tail = lines[resume:]
        if sum(len(l) + 1 for l in tail) < MIN_TAIL_CHARS:
            resume = None                    # tail too short to be worth keeping
        elif _tail_looks_like_refs(lines, resume):
            resume = None                    # the tail is still references: cut it too

    # The span actually deleted is [cut_line, removed_end)
    removed_end = end if resume is not None else len(lines)

    # Safety check: on a scrambled two-column layout, abandon the cut rather than
    # risk deleting body text.
    if _region_is_scrambled(lines, cut_line, removed_end):
        return md, [], "skipped-scrambled-layout"

    if resume is None:
        body = "\n".join(lines[:cut_line]).rstrip()
        refs = [l for l in lines[cut_line:] if l.strip()]
    else:
        body = "\n".join(lines[:cut_line] + lines[resume:]).rstrip()
        refs = [l for l in lines[cut_line:end] if l.strip()]
        how += "+tail-kept"

    return body, refs, how


# ---------------------------------------------------------------------------
# 2) Conversion (the pymupdf4llm call is embedded right here)
# ---------------------------------------------------------------------------

def _require_pymupdf4llm():
    try:
        import pymupdf4llm  # noqa: F401
        return pymupdf4llm
    except ImportError:
        sys.exit(
            "Missing dependency: pymupdf4llm. Install it first:\n"
            "  python -m venv .venv\n"
            "  .venv/bin/pip install pymupdf4llm\n"
        )


def _fallback_text(pdf: Path) -> str:
    """
    pymupdf4llm raises from the C layer on some PDFs (StructTree/xref problems).
    Fall back to plain per-page text extraction so no content is lost
    (at the cost of losing heading structure).
    """
    import pymupdf
    doc = pymupdf.open(str(pdf))
    try:
        return "\n\n".join(page.get_text() for page in doc)
    finally:
        doc.close()


def _looks_empty_or_broken(text: str) -> bool:
    if not text.strip():
        return True
    bad = text.count("\ufffd")                    # Unicode replacement character
    return bad / max(len(text), 1) > 0.05         # heavy garbling


def pdf_to_markdown(pdf: Path, pages: str | None,
                    ocr_lang: str, force_ocr: bool) -> tuple[str, bool]:
    """PDF -> Markdown. Returns (markdown, degraded_to_plain_text)."""
    pymupdf4llm = _require_pymupdf4llm()

    md = None
    try:
        kw: dict = {}
        if pages:
            sel: list[int] = []
            for part in pages.split(","):
                part = part.strip()
                if "-" in part:
                    a, b = part.split("-")
                    sel.extend(range(int(a) - 1, int(b)))
                else:
                    sel.append(int(part) - 1)
            kw["pages"] = sel
        if ocr_lang:
            kw["ocr_language"] = ocr_lang
        if force_ocr:
            kw["force_ocr"] = True

        md = pymupdf4llm.to_markdown(str(pdf), **kw)
    except Exception as e:
        print(f"    - pymupdf4llm failed ({type(e).__name__}); "
              f"falling back to plain-text extraction", file=sys.stderr)
        try:
            md = _fallback_text(pdf)
        except Exception as e2:
            raise RuntimeError(f"extraction failed: {e2}") from e2
        if not isinstance(md, str):
            md = str(md)
        md = md.strip()
        if not md:
            raise RuntimeError(
                "extraction produced no text (the PDF may be corrupt, "
                "or its xref table is unreadable)")
        return md, True

    if not isinstance(md, str):        # guard against pymupdf returning a non-str
        md = str(md)
    md = md.strip()

    # Empty or garbled but no exception: retry once with forced OCR.
    if _looks_empty_or_broken(md) and not force_ocr:
        print("    - output looks empty or garbled; retrying with forced OCR ...",
              file=sys.stderr)
        try:
            retry_kw = {"force_ocr": True}
            if ocr_lang:
                retry_kw["ocr_language"] = ocr_lang
            retry = pymupdf4llm.to_markdown(str(pdf), **retry_kw)
            if isinstance(retry, str) and not _looks_empty_or_broken(retry):
                return retry.strip(), False
        except Exception:
            pass

    if not md:
        raise RuntimeError(
            "extraction produced no text (the PDF may be corrupt, "
            "or its xref table is unreadable)")
    return md, False


# ---------------------------------------------------------------------------
# 3) One file: convert -> cut -> write
# ---------------------------------------------------------------------------

def est_tokens(text: str) -> int:
    """Rough token estimate: CJK ~1.5 chars/token, everything else ~4 chars/token."""
    cjk = sum(1 for ch in text if "\u4e00" <= ch <= "\u9fff")
    return int(cjk / 1.5 + (len(text) - cjk) / 4)


def process_one(pdf: Path, outdir: Path, *, pages: str | None, ocr_lang: str,
                force_ocr: bool, keep_refs: bool, write_refs: bool) -> dict:
    md, degraded = pdf_to_markdown(pdf, pages, ocr_lang, force_ocr)
    full_tokens = est_tokens(md)

    if keep_refs:
        body, refs, how = md, [], "kept"
    else:
        body, refs, how = cut_references(md)

    outdir.mkdir(parents=True, exist_ok=True)
    md_path = outdir / f"{pdf.stem}.md"
    md_path.write_text(body + "\n", encoding="utf-8")

    refs_path = None
    if refs and write_refs:
        refs_path = outdir / f"{pdf.stem}.refs.md"
        refs_path.write_text("\n".join(refs) + "\n", encoding="utf-8")

    body_tokens = est_tokens(body)
    return {
        "source_pdf": str(pdf),
        "markdown": str(md_path),
        "references_file": str(refs_path) if refs_path else None,
        "tokens_full": full_tokens,
        "tokens_body": body_tokens,
        "tokens_refs": full_tokens - body_tokens,
        "references_cut": bool(refs),
        "method": how,
        "degraded_text_only": degraded,
    }


# ---------------------------------------------------------------------------
# 4) CLI
# ---------------------------------------------------------------------------

def main() -> int:
    ap = argparse.ArgumentParser(
        description="PDF -> Markdown -> cut the reference list "
                    "(self-contained single-file edition)")
    ap.add_argument("input", help="a PDF file or a directory (searched recursively)")
    ap.add_argument("-o", "--out", default=None,
                    help="output directory; defaults to the input PDF's directory")
    ap.add_argument("--pages", help='page selection, e.g. "1-5,8"')
    ap.add_argument("--ocr-lang", default="eng", help="OCR language, default: eng")
    ap.add_argument("--force-ocr", action="store_true",
                    help="force OCR on every page (scanned PDFs / broken text layer)")
    ap.add_argument("--keep-refs", action="store_true",
                    help="keep the reference list (cut by default)")
    ap.add_argument("--no-refs-file", action="store_true",
                    help="do not write the separate .refs.md (written by default, "
                         "so citations can be checked later)")
    args = ap.parse_args()

    src = Path(args.input).expanduser()
    if not src.exists():
        print(f"input not found: {src}", file=sys.stderr)
        return 1

    pdfs = sorted(src.rglob("*.pdf")) if src.is_dir() else [src]
    if not pdfs:
        print(f"no PDF files found: {src}", file=sys.stderr)
        return 1

    tot_full = tot_body = 0
    ok = fail = 0
    records = []

    for i, pdf in enumerate(pdfs, 1):
        outdir = Path(args.out).expanduser() if args.out else pdf.parent
        try:
            rec = process_one(
                pdf, outdir,
                pages=args.pages, ocr_lang=args.ocr_lang,
                force_ocr=args.force_ocr, keep_refs=args.keep_refs,
                write_refs=not args.no_refs_file,
            )
        except Exception as e:
            fail += 1
            print(f"[{i}/{len(pdfs)}] {pdf.name[:44]:<46} FAILED: {e}",
                  file=sys.stderr)
            continue

        ok += 1
        records.append(rec)
        tot_full += rec["tokens_full"]
        tot_body += rec["tokens_body"]
        flag = f"cut({rec['method']})" if rec["references_cut"] else (
            f"not-cut({rec['method']})" if rec["method"]
            else "not-cut(no-references-found)")
        if rec["degraded_text_only"]:
            flag += " [text-only-fallback]"
        print(f"[{i}/{len(pdfs)}] {pdf.name[:44]:<46} "
              f"{rec['tokens_full']:>6} -> {rec['tokens_body']:>6} tok  {flag}",
              file=sys.stderr)

    if tot_full:
        pct = 100 - 100 * tot_body // tot_full
        print(f"\nok {ok} / failed {fail}; total {tot_full:,} -> {tot_body:,} "
              f"tokens ({pct}% saved)", file=sys.stderr)

    # Summary manifest (written even for a single file, so an agent can read state).
    if records:
        outdir = Path(args.out).expanduser() if args.out else pdfs[0].parent
        outdir.mkdir(parents=True, exist_ok=True)
        (outdir / "_solo_index.json").write_text(
            json.dumps(records, ensure_ascii=False, indent=2), encoding="utf-8")

    return 0 if fail == 0 else 2


if __name__ == "__main__":
    raise SystemExit(main())
````

---

## Appendix B - Verification script: `selftest.py`

Write this file next to `solo-pdf2md.py` and run it with the project's venv. It
exercises the reference-cutting logic directly, so it needs no PDF.

````python
#!/usr/bin/env python3
"""Self-test for solo-pdf2md's reference-cutting logic. No PDF required."""
import importlib.util
import pathlib
import sys

HERE = pathlib.Path(__file__).resolve().parent


def load_module():
    path = HERE / "solo-pdf2md.py"
    if not path.exists():
        sys.exit(f"FAIL: {path} not found")
    spec = importlib.util.spec_from_file_location("solo_pdf2md", path)
    mod = importlib.util.module_from_spec(spec)
    spec.loader.exec_module(mod)
    return mod


ENTRIES = "\n".join(
    f"{i}. Author{i}, A. et al. (20{10 + i:02d}). Title of study number {i}. "
    f"Journal of Testing, {i}(2), {100 + i}-{120 + i}."
    for i in range(1, 13)
)

# A realistically sized body. The tool deliberately refuses to cut when a
# candidate reference heading sits in the first 15% of the document (that is a
# misfire guard), so tests must use a document of realistic proportions rather
# than a three-line stub.
PARA = (
    "The circadian clock coordinates daily rhythms in physiology and behaviour. "
    "Disruption of these rhythms has been associated with metabolic disease, "
    "inflammation and impaired recovery from critical illness in both animal "
    "models and human observational cohorts. "
)
BODY = "\n\n".join(f"Section {i}. {PARA * 2}" for i in range(1, 9))


def test_heading_cut(mod):
    md = (
        "# Title\n\n"
        f"{BODY}\n\n"
        "## Results\n\nSome results text here.\n\n"
        "## References\n\n" + ENTRIES + "\n"
    )
    body, refs, how = mod.cut_references(md)
    assert how.startswith("heading"), f"expected a heading cut, got {how!r}"
    assert "Some results text here" in body, "results text was lost"
    assert f"Section 8. {PARA}" in body, "body text was lost"
    assert "Author1," not in body, "reference entries leaked into the body"
    assert len(refs) >= 10, f"expected >=10 reference lines, got {len(refs)}"
    print("PASS: heading cut")


def test_entry_run_cut(mod):
    md = (
        "# Title\n\n"
        f"{BODY}\n\n"
        "## Acknowledgements\n\nThanks to everyone who helped.\n\n" + ENTRIES + "\n"
    )
    body, refs, how = mod.cut_references(md)
    assert how.startswith("entry-run"), f"expected an entry-run cut, got {how!r}"
    assert "Thanks to everyone who helped" in body, "acknowledgements were lost"
    assert f"Section 8. {PARA}" in body, "body text was lost"
    assert "Author1," not in body, "reference entries leaked into the body"
    print("PASS: entry-run cut (no heading)")


def test_scrambled_layout_refused(mod):
    # A two-column scramble: body sections interleaved with the reference list.
    md = (
        "# Title\n\n"
        f"{BODY}\n\n"
        "## References\n\n"
        "1. Author, A. (2020). A study of things. Journal of Testing, 1(1), 1-10.\n\n"
        "## Concluding Remarks\n\nWe conclude something important.\n\n"
        "## Author Contributions\n\nEveryone contributed to this work.\n\n"
        "## Funding\n\nThis work was supported by grant number 12345.\n\n"
        "2. Author, B. (2021). Another study of things. Journal, 2(2), 20-30.\n"
        "3. Author, C. (2022). A third study of things. Journal, 3(3), 30-40.\n"
    )
    body, refs, how = mod.cut_references(md)
    assert how == "skipped-scrambled-layout", f"expected a refusal, got {how!r}"
    assert "We conclude something important" in body, "body text was deleted"
    assert "grant number 12345" in body, "funding text was deleted"
    assert refs == [], "nothing should have been extracted"
    print("PASS: scrambled layout is refused")


def test_no_references(mod):
    md = "# Title\n\n" + BODY + "\n"
    body, refs, how = mod.cut_references(md)
    assert how == "", f"expected no cut, got {how!r}"
    assert body == md, "body must be returned completely unchanged when nothing matches"
    assert refs == []
    print("PASS: no references leaves body untouched")


def main():
    mod = load_module()
    test_heading_cut(mod)
    test_entry_run_cut(mod)
    test_scrambled_layout_refused(mod)
    test_no_references(mod)
    print("ALL TESTS PASSED")


if __name__ == "__main__":
    main()
````

---

## Appendix C - License notice

`solo-pdf2md` is licensed **AGPL-3.0-or-later** and carries the SPDX header:

```
# SPDX-License-Identifier: AGPL-3.0-or-later
# Copyright (C) 2026 solo-pdf2md contributors
```

It depends on `pymupdf4llm`, which is dual-licensed under AGPL-3.0 or a
commercial license from Artifex Software. This project uses the AGPL-3.0 option,
which is why the project as a whole is AGPL-3.0.
