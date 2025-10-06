

# Data & File Cleanup with Python — updated Oct 6, 2025

Practical guide for **cleaning files and tabular data**. Covers renaming and deduping files, validating CSV/Excel schemas, fixing common issues (bad encodings, whitespace, mixed types), dropping or quarantining bad records, and writing clean outputs. Pairs with `013_CSVEXCEL.md` for deeper I/O and `016_LOGGING.md` for audit trails.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- You have input folders of files and spreadsheets needing cleanup.  
- Prefer non‑destructive workflows (write to `out/` and keep logs).

**Install**
```sh
pip install pandas openpyxl python-dotenv chardet "pyarrow>=17"
```
- `pandas` for tabular operations.  
- `openpyxl` for .xlsx.  
- `chardet` for encoding detection.  
- `pyarrow` for fast CSV/Parquet I/O.

---

## Table of Contents
- [0) Workflow and safety](#0-workflow-and-safety)
- [1) Detect and fix text encodings](#1-detect-and-fix-text-encodings)
- [2) File renaming and deduplication](#2-file-renaming-and-deduplication)
- [3) Load CSV/Excel safely](#3-load-csvexcel-safely)
- [4) Validate schema and required columns](#4-validate-schema-and-required-columns)
- [5) Clean values: whitespace, case, types](#5-clean-values-whitespace-case-types)
- [6) De‑duplicate rows](#6-de-duplicate-rows)
- [7) Remove or quarantine bad records](#7-remove-or-quarantine-bad-records)
- [8) Standardize dates and numbers](#8-standardize-dates-and-numbers)
- [9) Write cleaned outputs](#9-write-cleaned-outputs)
- [10) Mini CLI script](#10-mini-cli-script)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Workflow and safety
**What**: A repeatable, auditable cleanup flow.  
**Why**: Prevents accidental data loss and lets you reproduce results.
- **Never overwrite inputs**. Write cleaned files to `out/` and put rejects into `rejects/`.  
- Log counts: rows in/out, number dropped, duplicates removed.  
- Version outputs with timestamps, e.g., `clean_2025-10-06.csv`.

---

## 1) Detect and fix text encodings
**What**: CSVs arrive in mixed encodings; detect and re‑encode to UTF‑8.
```python
from pathlib import Path
import chardet

def to_utf8(path: Path, dest: Path):
    raw = path.read_bytes()
    enc = chardet.detect(raw)["encoding"] or "utf-8"
    text = raw.decode(enc, errors="replace")
    dest.write_text(text, encoding="utf-8")

inp = Path("data/vendor.csv"); out = Path("work/vendor_utf8.csv")
out.parent.mkdir(parents=True, exist_ok=True)
to_utf8(inp, out)
```

---

## 2) File renaming and deduplication
**What**: Normalize names and remove duplicate files by hash.
```python
from pathlib import Path
import hashlib, re, shutil

SRC = Path("incoming"); DEST = Path("staged"); DEST.mkdir(exist_ok=True)
seen = {}
for p in SRC.glob("**/*"):
    if not p.is_file():
        continue
    # compute hash to detect duplicates
    h = hashlib.sha256(p.read_bytes()).hexdigest()
    if h in seen:
        print("dup:", p, "=", seen[h])
        continue
    seen[h] = p
    # normalize filename: lowercase, spaces→_, remove unsafe chars
    base = re.sub(r"[^a-zA-Z0-9_.-]", "_", p.name.lower())
    dest = DEST / base
    # avoid collisions
    i = 1
    while dest.exists():
        stem, ext = dest.stem, dest.suffix
        dest = DEST / f"{stem}_{i}{ext}"; i += 1
    shutil.copy2(p, dest)
```
Notes:
- Use `copy2` for non‑destructive staging.  
- Keep a CSV log of `src → dest` if required for audits.

---

## 3) Load CSV/Excel safely
**What**: Read large, messy files without surprises.
```python
import pandas as pd

# CSV: infer dtypes lazily, keep strings as objects, avoid date auto-parse
df = pd.read_csv("work/vendor_utf8.csv", dtype=str, keep_default_na=False)

# Excel
xlsx = pd.read_excel("incoming/report.xlsx", sheet_name=0, dtype=str)
```
Tips:
- Use `dtype=str` initially to avoid silent numeric coercion. Clean later.  
- Set `na_filter=True` if you want pandas NA detection; otherwise keep empty strings.

---

## 4) Validate schema and required columns
**What**: Ensure mandatory columns exist with correct names; create a reject set otherwise.
```python
REQUIRED = {"id", "email", "created_at"}
missing = REQUIRED - set(df.columns)
if missing:
    raise ValueError(f"missing columns: {sorted(missing)}")
```
Optional: enforce allowed sets and column order.
```python
EXPECTED = ["id","email","created_at","amount"]
df = df.reindex(columns=EXPECTED, fill_value="")
```

---

## 5) Clean values: whitespace, case, types
**What**: Normalize obvious defects before type conversion.
```python
# trim whitespace across all string cells
for c in df.columns:
    df[c] = df[c].astype(str).str.strip()

# emails lowercase, strip invalid
df["email"] = df["email"].str.lower()

# remove zero‑width and non‑breaking spaces
df = df.replace({"\u200b": "", "\xa0": " "}, regex=True)
```
Type conversions with safe coercion:
```python
import pandas as pd
# numbers
df["amount"] = pd.to_numeric(df["amount"], errors="coerce")  # invalid → NaN
# booleans
TRUE = {"true","1","yes","y"}; FALSE = {"false","0","no","n"}

def to_bool(s: str):
    s = (s or "").strip().lower()
    return True if s in TRUE else False if s in FALSE else pd.NA

df["active"] = df.get("active", "").map(to_bool)
```

---

## 6) De‑duplicate rows
**What**: Remove exact duplicates and near‑dupes based on keys.
```python
# exact duplicate rows
before = len(df)
df = df.drop_duplicates()
print("drop exact dupes:", before - len(df))

# key‑based (same email + date)
df = df.sort_values(["email","created_at"])  # choose a deterministic keep
df = df.drop_duplicates(subset=["email","created_at"], keep="first")
```
Advanced: fuzzy dedupe with `thefuzz` or record‑linkage libraries when needed.

---

## 7) Remove or quarantine bad records
**What**: Keep outputs clean and store rejects for review.
```python
import pandas as pd
from pathlib import Path

rejects = []

# invalid email simple check
bad_email = ~df["email"].str.contains(r"^[^@\s]+@[^@\s]+\.[^@\s]+$", na=True)
rejects.append(df[bad_email].assign(reason="bad_email"))
df = df[~bad_email]

# missing id or created_at
bad_required = df["id"].eq("") | df["created_at"].eq("")
rejects.append(df[bad_required].assign(reason="missing_required"))
df = df[~bad_required]

REJ = Path("rejects"); REJ.mkdir(exist_ok=True)
if rejects:
    pd.concat(rejects, ignore_index=True).to_csv(REJ/"rejects.csv", index=False)
```

---

## 8) Standardize dates and numbers
**What**: Convert mixed formats to ISO 8601 and numeric types.
```python
import pandas as pd
# parse many date formats, localize if needed, output ISO date strings
s = pd.to_datetime(df["created_at"], errors="coerce", utc=True)
df["created_at"] = s.dt.strftime("%Y-%m-%dT%H:%M:%SZ")

# currency amounts → 2 decimals
df["amount"] = (df["amount"].astype(float).round(2))
```

---

## 9) Write cleaned outputs
**What**: Produce stable CSV/Parquet/Excel with explicit encoding and NA handling.
```python
from pathlib import Path
import pandas as pd

OUT = Path("out"); OUT.mkdir(exist_ok=True)
# CSV
(df.fillna(""))\
  .to_csv(OUT/"clean.csv", index=False, encoding="utf-8", line_terminator="\n")
# Parquet (fast, typed)
df.to_parquet(OUT/"clean.parquet")
# Excel
with pd.ExcelWriter(OUT/"clean.xlsx", engine="openpyxl") as xw:
    df.to_excel(xw, sheet_name="data", index=False)
```

---

## 10) Mini CLI script
**What**: One file you can adapt for most feeds.
```python
# file: clean_csv.py
import argparse, pandas as pd

REQUIRED = ["id","email","created_at"]

def run(src: str, dest: str):
    df = pd.read_csv(src, dtype=str, keep_default_na=False)
    # trim
    for c in df.columns:
        df[c] = df[c].astype(str).str.strip()
    # validate
    missing = set(REQUIRED) - set(df.columns)
    if missing:
        raise SystemExit(f"missing columns: {sorted(missing)}")
    # normalize
    df["email"] = df["email"].str.lower()
    df = df.drop_duplicates()
    df.to_csv(dest, index=False)

if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("src"); ap.add_argument("dest")
    run(**vars(ap.parse_args()))
```
Run:
```sh
python clean_csv.py incoming/raw.csv out/clean.csv
```

---

## 11) Troubleshooting
- **`UnicodeDecodeError`**: detect encoding first (Section 1) and re‑encode to UTF‑8.  
- **Columns shift on read**: bad delimiters or quoted commas; set `sep=","` and `quotechar='"'`, or use `engine="python"`.  
- **Excel date confusion**: Excel stores serial dates; read as `dtype=str` first and parse manually.  
- **Scientific notation in CSV**: write with `float_format='%.2f'` or cast to string before save.  
- **Large files slow**: use `usecols=...`, `chunksize=200_000`, or convert to Parquet.  
- **Fuzzy dedupe performance**: block by first letter/domain before fuzzy matching.

---

## 12) Recap
```plaintext
Stage files safely → normalize names and remove duplicate files → detect encoding → load as strings → validate schema → trim/normalize → convert types → dedupe rows → quarantine bad records → standardize dates/numbers → write clean CSV/Parquet/Excel with logs
```

**Next**: For downstream analytics see `011_GRAPHS.md`. For scheduled cleanups see `026_TASK_SCHEDULING.md`. For email/reporting of results use `014_EMAILS.md`. For parsing messy logs before cleanup, see `027_TEXT_PARSING.md`. 