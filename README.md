# solo-pdf2md

**PDF -> Markdown -> strip the references.** A single-file, self-contained CLI for
reading research papers without paying context-window tokens for a bibliography
you are not going to read.

```
PDF  -->  layout-aware Markdown  -->  cut the reference list  -->  body.md + refs.md
         (PyMuPDF4LLM)                (this tool's own logic)
```

Everything runs locally. No API keys, no uploads, no per-page cost.

> **Setting this up with an AI agent?** Hand it
> [`AI-INSTALL-GUIDE.md`](AI-INSTALL-GUIDE.md) - a single self-contained file
> containing the full instructions *and* the complete source code, so an agent
> can install and verify everything in one pass.

---

## Why

In a typical research paper the reference list is **roughly 10-50% of the whole
document**. If you feed the full text to an LLM for reading or analysis, you
spend a large share of your context on entries the model does not need.

Measured on a real 41-paper corpus (circadian-biology literature, 683k tokens
total): stripping references saved **19%** overall, and up to **50%** on
individual papers - while **never deleting any body text** (see
[Safety](#safety-never-eat-the-body)).

## What it does

1. **Convert** - `pymupdf4llm` reads the PDF with a layout model, so multi-column
   papers keep the correct reading order and headings come out as `#` / `##` /
   `###`. Scanned or garbled pages trigger Tesseract OCR automatically.
2. **Cut** - the reference section is located in the *Markdown text* (not in the
   PDF, where regex cannot work) and removed.
3. **Write** - two files: the body, and the references kept separately so you can
   still check a citation on demand.

## Install

```bash
python -m venv .venv
.venv/bin/pip install -r requirements.txt
```

Requires Python 3.10+. For OCR of scanned PDFs, install
[Tesseract](https://github.com/tesseract-ocr/tesseract) (`brew install tesseract`).
Without it, text-layer extraction still works.

## Usage

```bash
# one PDF, output next to the input
python solo-pdf2md.py paper.pdf

# choose an output directory
python solo-pdf2md.py paper.pdf -o out/

# a whole directory, recursively
python solo-pdf2md.py /path/to/pdfs -o out/

# keep the references (default is to cut them)
python solo-pdf2md.py paper.pdf --keep-refs

# selected pages / force OCR
python solo-pdf2md.py paper.pdf --pages 1-5,8
python solo-pdf2md.py scan.pdf --ocr-lang eng --force-ocr
```

| Flag | Meaning |
|---|---|
| `-o, --out` | Output directory (default: next to the input PDF) |
| `--pages` | Page selection, e.g. `"1-5,8"` |
| `--ocr-lang` | Tesseract language, default `eng` |
| `--force-ocr` | OCR every page (scanned / broken text layer) |
| `--keep-refs` | Do not cut the reference list |
| `--no-refs-file` | Do not write the separate `.refs.md` |

## Output

For `paper.pdf` you get:

| File | Contents |
|---|---|
| `paper.md` | The body, references removed |
| `paper.refs.md` | The reference list on its own |
| `_solo_index.json` | Per-file summary: token counts, method used, warnings |

Each run prints a per-file report:

```
[1/1] 2017_022_A_non_transcriptional_role_for_the_    13804 -> 11676 tok  cut(heading+tail-kept)

ok 1 / failed 0; total 13,804 -> 11,676 tokens (16% saved)
```

The `method` field tells you exactly what happened:

| `method` | Meaning |
|---|---|
| `heading` | Cut at a `References` / `Bibliography` heading |
| `entry-run` | No heading; cut at a detected run of numbered entries |
| `...+tail-kept` | Content after the reference list (Methods, figures) was preserved |
| `kept` | `--keep-refs` was used |
| `skipped-scrambled-layout` | Safety guard tripped; nothing was cut |
| *(empty)* | No reference list was found |

## How the reference detection works

Three tiers, in order:

1. **Heading** - matches `References` / `Bibliography` / `Literature Cited` /
   `Works Cited`, tolerating Markdown `#`, bold markers, leading numbering and a
   trailing colon. When several candidates exist, the one actually followed by
   the most entries wins. Handles the `**References and Notes:**` form, where the
   colon sits *inside* the bold markers.
2. **Entry run** - many publishers (PNAS, JCI, Nature journals) print no
   "References" heading at all; the list just starts. This tier finds the longest
   run of numbered entries carrying citation features (year, `et al.`, DOI, page
   ranges), tolerating bullet prefixes and PDF line numbers:

   ```
   - 476 1. Fernandez-Mendoza, J. & Vgontzas, A.N. ...
   ```
3. **Backward extension** - a reference list broken by figure captions or page
   footers can be split in two. After locating a run, the cut point walks
   backwards to pull in the earlier half.

## Safety: never eat the body

Cutting by position is dangerous, so three guards run before anything is removed:

1. **Cut only the list, keep what follows.** Many journals place Methods, figure
   legends or supplementary tables *after* the bibliography. The tool finds the
   next structural marker and keeps everything from there on.
2. **Reject a reference-heavy tail.** If the text it was about to keep is itself
   mostly citation entries, it is still the reference list - cut it too.
3. **Refuse on scrambled layouts.** If body headings (`Methods`, `Funding`, ...)
   appear *inside* the region about to be deleted, the PDF is two-column with
   broken reading order and a positional cut would destroy real content. The tool
   then **skips the cut entirely** and reports `skipped-scrambled-layout`.

Guard 3 is why the headline saving is 19% rather than 26%: an earlier, more
aggressive version reached 26% by silently deleting Methods and figure legends.
Correctness was chosen over the extra points.

## Known limitations

- **Two-column PDFs with badly scrambled reading order** may be skipped rather
  than cut. The report says so explicitly; nothing is lost.
- **Corrupt PDFs** (broken xref tables) fail with a clear error instead of
  producing an empty file.
- **CID-font PDFs** (common in older Chinese typesetting) can yield garbled
  glyphs. Use `--force-ocr`.
- Token counts are estimates (`~4 chars/token` for Latin text, `~1.5` for CJK),
  not a real tokenizer.
- The reference-cut heuristics are tuned on **English-language life-science and
  medical papers**. Other fields and languages may need adjustment.

## Files

| File | Purpose |
|---|---|
| `solo-pdf2md.py` | The tool (single file, no local imports) |
| `AI-INSTALL-GUIDE.md` | Self-contained guide + source for handing to an AI agent |
| `README.md` | This file |
| `THIRD_PARTY_LICENSES.md` | Dependency licenses and why this project is AGPL |
| `requirements.txt` | Python dependencies |

## License

This project is released under the **GNU Affero General Public License v3.0**
(see [LICENSE](LICENSE)).

It depends on [PyMuPDF4LLM](https://github.com/pymupdf/PyMuPDF4LLM), which is
**dual-licensed: AGPL-3.0 or a commercial license from Artifex Software**. This
project uses it under the AGPL-3.0 option, so the project as a whole is
distributed under AGPL-3.0. If you need to use this code in a proprietary
product or behind a public network service, you must obtain a commercial license
from [Artifex](https://artifex.com/licensing).

See [THIRD_PARTY_LICENSES.md](THIRD_PARTY_LICENSES.md) for the full dependency list.

## Acknowledgments

Layout analysis and PDF parsing are provided by
[PyMuPDF4LLM](https://github.com/pymupdf/PyMuPDF4LLM) and
[PyMuPDF](https://github.com/pymupdf/PyMuPDF) (Artifex Software).
OCR is performed by [Tesseract](https://github.com/tesseract-ocr/tesseract).
The reference-detection and cut-safety logic in this project is original work.
