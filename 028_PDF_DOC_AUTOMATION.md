

# PDF Document Automation with Python — updated Oct 6, 2025

Practical guide to automating PDF workflows: **extracting text**, **splitting/merging**, **reordering**, **annotating/highlighting**, **redacting**, **filling forms**, and **digitally signing**. Uses modern Python libraries and, where appropriate, small CLI helpers. Pair with `002_SETUP.md` for venv and with `010_OCR.md` if pages need OCR before extraction.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Input PDFs are not DRM‑restricted. Encrypted PDFs require passwords and may limit operations.

**Install**
```sh
pip install pypdf pdfminer.six pymupdf reportlab pyhanko
# optional utilities
pip install pdfplumber "pikepdf>=9"
# optional CLI for OCR and optimization
brew install ocrmypdf qpdf
```

> Libraries at a glance:
> - **pypdf**: merge/split/reorder, metadata, forms, encryption.  
> - **pdfminer.six/pdfplumber**: rich text extraction and layout analysis.  
> - **PyMuPDF (fitz)**: annotations, redaction, images, rendering.  
> - **pikepdf**: low‑level PDF (qpdf bindings), object fixes.  
> - **pyHanko**: create and validate **digital signatures**.  
> - **ocrmypdf (CLI)**: OCR and text‑searchable PDFs.

---

## Table of Contents
- [0) Pick the right tool](#0-pick-the-right-tool)
- [1) Read basics: metadata, pages, encryption](#1-read-basics-metadata-pages-encryption)
- [2) Extract text](#2-extract-text)
- [3) Split, merge, and reorder pages](#3-split-merge-and-reorder-pages)
- [4) Forms: read, fill, and flatten](#4-forms-read-fill-and-flatten)
- [5) Annotations and highlights](#5-annotations-and-highlights)
- [6) Redaction (remove text/graphics)](#6-redaction-remove-textgraphics)
- [7) Digital signatures (PKCS#7)](#7-digital-signatures-pkcs7)
- [8) Images: extract and add](#8-images-extract-and-add)
- [9) Optimize, compress, and repair](#9-optimize-compress-and-repair)
- [10) OCR when text is not selectable](#10-ocr-when-text-is-not-selectable)
- [11) Automating end-to-end workflows](#11-automating-end-to-end-workflows)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Recap](#13-recap)

---

## 0) Pick the right tool
**What**: Choose APIs by task.  
**Why**: PDFs mix text, vector graphics, and images; each tool excels at different layers.
- Simple page manipulation → **pypdf**.  
- Accurate text extraction (layout aware) → **pdfminer.six** or **pdfplumber**.  
- Visual edits, annotations, redaction, rendering → **PyMuPDF**.  
- Signing/validation → **pyHanko**.  
- Corrupted/complex objects → **pikepdf/qpdf**.

---

## 1) Read basics: metadata, pages, encryption
**What**: Inspect document for page count, metadata, and encryption to decide next steps.
```python
from pypdf import PdfReader

reader = PdfReader("input.pdf")
print("pages:", len(reader.pages))
print("metadata:", reader.metadata)
print("is_encrypted:", reader.is_encrypted)
```
Decrypt (if you know the password):
```python
if reader.is_encrypted:
    reader.decrypt("password123")
```

---

## 2) Extract text
**What**: Get machine‑readable text.  
**Why**: Power search, classification, and exports.

### 2.1 Quick text via pypdf
```python
from pypdf import PdfReader
reader = PdfReader("input.pdf")
text = "\n".join(page.extract_text() or "" for page in reader.pages)
open("out.txt","w",encoding="utf-8").write(text)
```

### 2.2 Detailed layout via pdfminer.six
```python
from pdfminer.high_level import extract_text
text = extract_text("input.pdf")
print(text[:500])
```

### 2.3 With pdfplumber (tables, char boxes)
```python
import pdfplumber
with pdfplumber.open("input.pdf") as pdf:
    first = pdf.pages[0]
    text = first.extract_text()
    tables = first.extract_tables()
    print(text)
    print(tables)
```

> If text is empty, see Section 10 for OCR.

---

## 3) Split, merge, and reorder pages
**What**: Assemble packets, remove junk, and standardize order.

### 3.1 Split selected pages
```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("input.pdf")
writer = PdfWriter()
for i in [0,2,5]:  # zero‑based pages
    writer.add_page(reader.pages[i])
with open("subset.pdf","wb") as f:
    writer.write(f)
```

### 3.2 Merge multiple PDFs
```python
from pypdf import PdfMerger
merger = PdfMerger()
for path in ["a.pdf","b.pdf","c.pdf"]:
    merger.append(path)
merger.write("merged.pdf"); merger.close()
```

### 3.3 Rotate, scale, and reorder
```python
from pypdf import PdfReader, PdfWriter
from pypdf.generic import RectangleObject

r = PdfReader("input.pdf"); w = PdfWriter()
for p in r.pages:
    p.rotate(90)
    # optional: set a new mediabox (crop)
    p.mediabox = RectangleObject([0, 0, p.mediabox.width, p.mediabox.height])
    w.add_page(p)
w.write(open("rotated.pdf","wb"))
```

---

## 4) Forms: read, fill, and flatten
**What**: Programmatically complete form fields and lock them.
```python
from pypdf import PdfReader, PdfWriter

r = PdfReader("form.pdf")
w = PdfWriter()
w.append(r)
fields = r.get_form_text_fields()  # dict of field name → value
print(fields)

w.update_page_form_field_values(w.pages[0], {
    "name": "Ned Perkins",
    "date": "2025-10-06",
})
# flatten to make entries non‑editable
w.flatten_annotations()
with open("form_filled.pdf","wb") as f:
    w.write(f)
```

---

## 5) Annotations and highlights
**What**: Add comments, highlights, and links.
```python
import fitz  # PyMuPDF

with fitz.open("input.pdf") as doc:
    page = doc[0]
    # highlight the first occurrence of text
    areas = page.search_for("Important")
    if areas:
        annot = page.add_highlight_annot(areas[0])
        annot.set_info(content="Check this.")
    # add a sticky note
    page.add_text_annot((72, 72), "Review by Friday")
    doc.save("annotated.pdf")
```

---

## 6) Redaction (remove text/graphics)
**What**: Permanently remove sensitive content.  
**Why**: Hiding with a black box is not redaction; content must be deleted from the content stream.
```python
import fitz  # PyMuPDF

with fitz.open("input.pdf") as doc:
    p = doc[0]
    for rect in p.search_for("SECRET"):
        p.add_redact_annot(rect, fill=(0,0,0))
    p.apply_redactions()
    doc.save("redacted.pdf")
```
Validate by searching text in the output and by opening with a text editor to ensure data is gone.

---

## 7) Digital signatures (PKCS#7)
**What**: Create a cryptographic signature covering the document or a visible widget.  
**Why**: Integrity, authenticity, and non‑repudiation.

### 7.1 Create a self‑signed key (demo)
```sh
openssl req -x509 -newkey rsa:2048 -keyout sk.pem -out cert.pem -days 365 -nodes -subj "/CN=Ned Demo"
```

### 7.2 Sign with pyHanko
```python
from pyhanko.sign import signers
from pyhanko_certvalidator import ValidationContext

with open("cert.pem","rb") as f:
    cert = f.read()
signer = signers.SimpleSigner.load("sk.pem", cert, key_passphrase=None)

with open("input.pdf","rb") as inf, open("signed.pdf","wb") as outf:
    pdf_out = signers.sign_pdf(inf, signer=signer)
    outf.write(pdf_out.getbuffer())
```
Validation: use Adobe Reader or `pyhanko` CLI (`pyhanko validate signed.pdf`). For production, use a CA‑issued cert or an AATL‑trusted signer, and consider using a TSA for timestamps.

---

## 8) Images: extract and add
**What**: Pull embedded images or stamp new ones.

### 8.1 Extract images
```python
import fitz
with fitz.open("input.pdf") as doc:
    for i, page in enumerate(doc):
        for img in page.get_images(full=True):
            xref = img[0]
            pix = fitz.Pixmap(doc, xref)
            if pix.alpha:  # remove alpha for saving as PNG/JPG
                pix = fitz.Pixmap(fitz.csRGB, pix)
            pix.save(f"page{i+1}_img{xref}.png")
```

### 8.2 Add a watermark/stamp
```python
import fitz
with fitz.open("input.pdf") as doc, fitz.open() as stamp:
    p = stamp.new_page(width=400, height=100)
    p.insert_text((20,60), "CONFIDENTIAL", fontsize=36, color=(1,0,0), rotate=0)
    for page in doc:
        page.show_pdf_page(page.rect, stamp, 0, overlay=True)
    doc.save("stamped.pdf")
```

---

## 9) Optimize, compress, and repair
**What**: Reduce size, linearize for web, and fix broken structures.

### 9.1 Optimize with qpdf
```sh
qpdf --linearize input.pdf optimized.pdf
```

### 9.2 Re‑write with pikepdf
```python
import pikepdf
with pikepdf.open("input.pdf") as pdf:
    pdf.save("rewritten.pdf", linearize=True)
```
Notes:
- True image compression requires re‑encoding images; consider Ghostscript or PyMuPDF rasterization for heavy compression, at the cost of text/selectability.

---

## 10) OCR when text is not selectable
**What**: Scanned PDFs need OCR to extract text.
```sh
# creates a text-searchable PDF; preserves original images
ocrmypdf input.pdf ocr.pdf --language eng --optimize 0
```
Then run text extraction on `ocr.pdf`. See `010_OCR.md` for Python OCR patterns.

---

## 11) Automating end-to-end workflows
**What**: Compose steps into a reliable pipeline.
```python
# file: pdf_pipeline.py
from pypdf import PdfReader, PdfWriter
import fitz

# 1) Split first 3 pages
r = PdfReader("input.pdf"); w = PdfWriter()
for i in range(3):
    w.add_page(r.pages[i])
with open("part.pdf","wb") as f: w.write(f)

# 2) Annotate first page
with fitz.open("part.pdf") as doc:
    doc[0].add_text_annot((72,72), "QA reviewed")
    doc.save("part_annot.pdf")

# 3) Merge with an appendix
from pypdf import PdfMerger
m = PdfMerger(); m.append("part_annot.pdf"); m.append("appendix.pdf"); m.write("bundle.pdf"); m.close()
```
Tips:
- Log each step and keep intermediates in `/tmp` or a `build/` folder.  
- Validate output size, page count, and text presence before sending.

---

## 12) Troubleshooting
- **Extraction returns empty text**: likely scanned; run OCR (Section 10).  
- **`PdfReadError` / malformed file**: try `pikepdf` open/save, then re‑process.  
- **Annotations not visible**: ensure you saved and the viewer displays comments; try flattening where appropriate.  
- **Redaction not permanent**: you must `apply_redactions()` and re‑open to confirm.  
- **Signing fails/invalid**: check certificate chain and time; use a TSA or an AATL‑trusted cert in production.  
- **Form fill not persistent**: call `flatten_annotations()` after setting values.

---

## 13) Recap
```plaintext
Inspect → extract text (or OCR) → split/merge/reorder → fill/annotate/redact → sign → optimize/repair → validate outputs
```

**Next**: Feed extracted data into `013_CSVEXCEL.md`, schedule pipelines via `026_TASK_SCHEDULING.md`, and notify results using `031_NOTIFICATION_AUTOMATION.md`. For scanning/OCR specifics, see `010_OCR.md`. 