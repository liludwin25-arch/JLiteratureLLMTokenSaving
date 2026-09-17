# Third-Party Licenses

`solo-pdf2md` is distributed under **AGPL-3.0** (see [LICENSE](LICENSE)).
It relies on the following third-party components. **No third-party source code
is vendored into this repository** - everything is declared in
`requirements.txt` and installed by the user.

## Direct dependency

| Component | License | Notes |
|---|---|---|
| [PyMuPDF4LLM](https://github.com/pymupdf/PyMuPDF4LLM) | **AGPL-3.0** or Artifex Commercial | Dual-licensed. Used here under AGPL-3.0. |

`pymupdf4llm` pulls in the following transitively:

| Component | License | Notes |
|---|---|---|
| [PyMuPDF](https://github.com/pymupdf/PyMuPDF) | **AGPL-3.0** or Artifex Commercial | Dual-licensed. |
| [PyMuPDF Layout](https://github.com/ArtifexSoftware/pymupdf_layout) | **AGPL-3.0** or Artifex Commercial | Dual-licensed. Became AGPL-3.0 as of 1.28.2; earlier releases were PolyForm Noncommercial. This is why `requirements.txt` pins `>=1.28.2`. |
| [MuPDF](https://mupdf.com/) | **AGPL-3.0** or Artifex Commercial | The underlying C engine. |
| [onnxruntime](https://github.com/microsoft/onnxruntime) | MIT | Layout model runtime. |
| [NumPy](https://numpy.org/) | BSD-3-Clause | |
| [NetworkX](https://networkx.org/) | BSD-3-Clause | |
| [protobuf](https://github.com/protocolbuffers/protobuf) | BSD-3-Clause | |
| [FlatBuffers](https://github.com/google/flatbuffers) | Apache-2.0 | |
| [PyYAML](https://pyyaml.org/) | MIT | |
| [tabulate](https://github.com/astanin/python-tabulate) | MIT | |
| [psutil](https://github.com/giampaolo/psutil) | BSD-3-Clause | |
| [packaging](https://github.com/pypa/packaging) | Apache-2.0 / BSD-2-Clause | |

## Optional external tool

| Component | License | Notes |
|---|---|---|
| [Tesseract OCR](https://github.com/tesseract-ocr/tesseract) | Apache-2.0 | Only needed for scanned PDFs (`--force-ocr`, or automatic OCR of image-only pages). Not required for text-layer PDFs. |

## Why this project is AGPL-3.0

`pymupdf4llm` and the PyMuPDF stack are strong-copyleft (AGPL-3.0) unless you buy
a commercial license. Because `solo-pdf2md.py` imports and links against that
library, the combined work must be distributed under AGPL-3.0. Artifex's own
guidance states that building an open-source project under AGPL-compatible terms
is the free path; proprietary products and public network services require a
commercial license.

If you need this code under permissive terms, your options are:

1. Obtain a commercial license from [Artifex](https://artifex.com/licensing), or
2. Replace the PDF engine with a permissively-licensed library (e.g. pdfplumber,
   MIT; pypdfium2, BSD-3-Clause) - accepting the loss of the layout model, which
   is what keeps multi-column reading order correct.

## Attribution in the source

The `solo-pdf2md.py` file carries an SPDX identifier:

```
# SPDX-License-Identifier: AGPL-3.0-or-later
# Copyright (C) 2026 solo-pdf2md contributors
```

No code from PyMuPDF4LLM or PyMuPDF has been copied into this project; they are
used exclusively through their public Python API.
