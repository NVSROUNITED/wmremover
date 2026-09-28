[README.md](https://github.com/user-attachments/files/32758876/README.md)
# wmremover# wmremover

[![CI](https://github.com/NVSRO/wmremover/actions/workflows/ci.yml/badge.svg)](https://github.com/NVSRO/wmremover/actions/workflows/ci.yml)
![Python](https://img.shields.io/badge/python-3.9%20%7C%203.10%20%7C%203.11%20%7C%203.12%20%7C%203.13-blue)
[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)

Remove watermarks from **PDF**, **DOCX** and **image** files (`.png`, `.jpg`,
`.tiff`, `.webp`…) from the command line. It runs one file or a whole folder and needs no GUI.

Watermarks are added in very different ways depending on the file type, so
`wmremover` picks a different strategy for each. It edits the document
structure when it can (lossless) and only rebuilds pixels when it has to.

```bash
wmremover pdf   report.pdf -o clean.pdf --text CONFIDENTIAL DRAFT
wmremover pdf   scan.pdf   -o clean.pdf --raster --region 450,20,120,40
wmremover docx  memo.docx  -o clean.docx
wmremover image photo.jpg  -o clean.jpg --region 20,20,180,60
wmremover auto  ./docs     -o ./cleaned --recursive
```

---

## How it works

| Type | Approach | Lossless? |
|------|----------|-----------|
| **PDF** (default) | Deletes watermark **annotations**, switches off watermark **layers**, **redacts** text stamps you name (e.g. `CONFIDENTIAL`), and can remove **logos repeated across pages**. | Yes. Real text stays selectable. |
| **PDF** `--raster` | For watermarks **baked into the page**: renders each page, **inpaints** the watermark area, and rebuilds the page. Pages you don't select are copied unchanged. | No. Rebuilt pages become images. |
| **DOCX** | Rewrites the Office XML directly: strips **WordArt/VML watermark shapes** from headers and removes the **page background**. Everything else is left byte-for-byte intact. | Yes. |
| **Image** | Rebuilds the covered pixels with **OpenCV inpainting**, using a rectangle, a mask you paint, or auto-detection of faint overlays. | No. Hidden pixels are *estimated*. |

**Rule of thumb for PDFs:** try the default mode first. If the watermark is
still there, it's baked in, so use `--raster`.

---

## Install

```bash
git clone https://github.com/NVSRO/wmremover.git
cd wmremover
pip install -e .
```

Requires Python 3.9+. Dependencies: PyMuPDF, lxml, OpenCV (headless), NumPy.

---

## Usage

### PDF: structural mode (lossless)

```bash
# Annotations + layers are always cleaned; --text redacts text stamps
wmremover pdf report.pdf -o clean.pdf --text CONFIDENTIAL "DO NOT COPY"

# Also drop a logo image that repeats on 3+ pages
wmremover pdf report.pdf -o clean.pdf --images --min-repeat 3
```

| Option | Meaning |
|---|---|
| `--text TERM …` | Text watermark strings to redact. |
| `--images` | Remove raster images repeated across pages. |
| `--min-repeat N` | Pages an image must appear on to count as repeated (default `2`). |

### PDF: raster mode (for baked-in watermarks)

```bash
# Inpaint a corner logo on every page
wmremover pdf scan.pdf -o clean.pdf --raster --region 450,20,120,40

# Two areas, only on pages 1-3, at print resolution
wmremover pdf scan.pdf -o clean.pdf --raster \
    --region 450,20,120,40 --region 40,800,200,20 --pages 1-3 --dpi 300

# Faint diagonal "DRAFT" across the page: let it auto-detect
wmremover pdf scan.pdf -o clean.pdf --raster --auto-mask
```

| Option | Meaning |
|---|---|
| `--raster` | Turn on raster mode. Needs `--region`, `--mask` or `--auto-mask`. |
| `--region X,Y,W,H` | Area in **PDF points** (1/72 inch, origin top-left). Repeatable. |
| `--mask FILE` | Painted mask image (white = remove), stretched to each page. |
| `--auto-mask` | Auto-detect faint, grey, see-through overlays. |
| `--pages SPEC` | Pages to rebuild, e.g. `1,3-5` (default: all). |
| `--dpi N` | Render resolution (default `150`; use `300` for print). |
| `--quality N` | JPEG quality of rebuilt pages (default `90`). |

> **Finding coordinates:** an A4 page is 595 × 842 points and US Letter is
> 612 × 792. Measure in any PDF viewer that shows cursor position in points,
> or estimate: a logo in the top-right corner of A4 is roughly `450,20,130,50`.

Rebuilt pages lose selectable text. If you need it back, run OCR afterwards
(for example `ocrmypdf clean.pdf clean_ocr.pdf`).

### DOCX

```bash
wmremover docx memo.docx -o clean.docx
```

No options needed. It finds and removes header watermark shapes and the page background automatically.

### Image

```bash
wmremover image scan.jpg -o clean.jpg --region 20,20,180,60   # rectangle, in pixels
wmremover image scan.jpg -o clean.jpg --mask mask.png         # white = remove
wmremover image scan.jpg -o clean.jpg --auto-mask             # faint overlays
```

`--radius N` sets the inpaint radius (default `3`). `--method telea|ns` chooses
the algorithm (default `telea`). With no `--region` or `--mask`, auto-detection is used.

### Batch

```bash
wmremover auto ./docs -o ./cleaned --recursive --text CONFIDENTIAL
```

This detects each file's type by extension and applies the matching handler. Output goes to
`./cleaned/name_clean.ext`. If `-o` is omitted, each cleaned file is written next to its original.
The output folder is never re-scanned, even if it sits inside the input folder.

You can overwrite a file in place by passing the same path to `-o`. Files are
written via a temp file, so a crash never leaves a half-written output.

---

## Python API

```python
from wmremover import (
    remove_pdf_watermarks,   # structural, lossless
    rasterize_pdf,           # raster fallback
    remove_docx_watermarks,
    remove_image_watermark,
)

remove_pdf_watermarks("in.pdf", "out.pdf", text_terms=["CONFIDENTIAL"])
rasterize_pdf("scan.pdf", "out.pdf", regions=[(450, 20, 120, 40)], pages="1-3")
remove_docx_watermarks("in.docx", "out.docx")
remove_image_watermark("in.jpg", "out.jpg", region=(20, 20, 180, 60))
```

Each call returns a small dict describing what was removed.

---

## Limitations

- **Image and raster inpainting is an estimate.** The pixels under a watermark are
  gone. Results are best on faint marks over plain backgrounds and weakest on
  opaque marks over detailed areas like photos or dense text.
- **Auto-detection is conservative.** It's tuned for light-grey see-through
  overlays. For anything else, use `--region` or `--mask`.
- **DOCX detection is heuristic.** It targets Word's standard watermark
  feature. A watermark pasted in as an ordinary picture in the body won't be detected.

---

## Development

```bash
pip install -e ".[dev]"
pytest              # 47 tests; every fixture is generated in code
ruff check . && ruff format --check .
```

CI runs lint and tests on Python 3.9–3.13 (Linux), plus Windows and macOS, and
builds the package on every push and pull request.

---

## Responsible use

This tool is for legitimate cases: removing your own watermarks, cleaning up
drafts, or stripping "SAMPLE"/"DRAFT" stamps from documents you own or have the
right to modify. Removing watermarks to strip credit from someone else's work or to get around copyright
may be illegal where you live. You are responsible for how you use it.

## License

MIT, see [LICENSE](LICENSE).
