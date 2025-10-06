

# Text Parsing with Python (Regex + Strings) — updated Oct 6, 2025

Practical guide to parsing logs, configs, and mixed text using Python **string methods** and the **`re`** module. Covers common patterns, flags, capturing groups, greedy vs lazy, multiline records, streaming large files, structured snippets (JSON/YAML/TOML) extraction, and robustness tips. Use with `002_SETUP.md` for environment setup.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Files are UTF‑8 unless noted.  
- You want reliable, copy‑pasteable patterns for real logs and configs.

**Install (optional helpers)**
```sh
pip install regex python-dateutil chardet ruamel.yaml tomli tomlkit orjson
```
- `regex`: enhanced regex engine (optional; stdlib `re` is sufficient for most tasks).  
- `python-dateutil`: flexible timestamp parsing.  
- `chardet`: guess file encodings.  
- `ruamel.yaml`, `tomli`/`tomlkit`, `orjson`: parse structured fragments.

---

## Table of Contents
- [0) Picking the right tool](#0-picking-the-right-tool)
- [1) Fast wins with string methods](#1-fast-wins-with-string-methods)
- [2) Regex basics: patterns, groups, flags](#2-regex-basics-patterns-groups-flags)
- [3) Common log parsing recipes](#3-common-log-parsing-recipes)
- [4) Multiline entries and record boundaries](#4-multiline-entries-and-record-boundaries)
- [5) Greedy vs lazy and backtracking](#5-greedy-vs-lazy-and-backtracking)
- [6) Named groups and data classes](#6-named-groups-and-data-classes)
- [7) Streaming large files safely](#7-streaming-large-files-safely)
- [8) Extracting embedded JSON/YAML/TOML](#8-extracting-embedded-jsonyamltoml)
- [9) Timestamps, timezones, and sorting](#9-timestamps-timezones-and-sorting)
- [10) Encodings and Unicode](#10-encodings-and-unicode)
- [11) Validation and error handling](#11-validation-and-error-handling)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Recap](#13-recap)

---

## 0) Picking the right tool
**What**: Choose strings or regex based on pattern complexity.  
**Why**: Reduces bugs and improves speed.
- Use **string methods** (`split`, `partition`, `startswith`, `find`) for simple delimited data.  
- Use **`re`** for variable spacing, optional fields, or complex validation.  
- Use **parsers** for structured blocks (JSON/YAML/TOML) instead of regex.

---

## 1) Fast wins with string methods
**What**: Parse fixed‑format lines without regex.  
**Why**: Faster and clearer.
```python
line = "2025-10-06 12:34:56 INFO app: user=ned action=login"
# split once on spaces, then key/value pairs
head, _space, rest = line.partition(" INFO ")
date, _space, time = head.partition(" ")
parts = dict(kv.split("=", 1) for kv in rest.split() if "=" in kv)
print(date, time, parts["user"], parts.get("action"))
```
Tips:
- Use `partition`/`rpartition` to avoid over‑splitting.  
- `split(maxsplit=n)` to limit work.  
- `removeprefix`/`removesuffix` to clean fields.

---

## 2) Regex basics: patterns, groups, flags
**What**: Build and apply regular expressions for flexible matching.
```python
import re
pat = re.compile(r"^(?P<ts>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+(?P<level>INFO|WARN|ERROR)\s+(?P<mod>\w+):\s+user=(?P<user>\w+)")
line = "2025-10-06 12:34:56 INFO app: user=ned action=login"
if m := pat.search(line):
    print(m.group("ts"), m.group("level"), m.group("user"))
```
Flags:
- `re.I` case‑insensitive, `re.M` multi‑line `^$`, `re.S` dot matches newline, `re.X` verbose.
Apply flags:
```python
pat = re.compile(r"^error: (.+)$", re.I|re.M)
```

---

## 3) Common log parsing recipes
**What**: Ready patterns for typical logs.

### 3.1 Nginx/Apache combined log
```python
import re
log_re = re.compile(r"""
^(?P<ip>\S+) \S+ \S+ \[(?P<ts>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d{3}) (?P<size>\S+)
""", re.X)
```

### 3.2 Key=value pairs (order may vary)
```python
import re
kv = dict(re.findall(r"(\w+)=([^\s]+)", "user=ned action=login resp=200"))
```

### 3.3 IPv4/IPv6, email, UUID
```python
ip_re = r"(?:(?:\d{1,3}\.){3}\d{1,3}|[A-Fa-f0-9:]+)"
email_re = r"[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}"
uuid_re = r"[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[1-5][0-9a-fA-F]{3}-[89abAB][0-9a-fA-F]{3}-[0-9a-fA-F]{12}"
```

### 3.4 Extract numbers and units
```python
import re
m = re.findall(r"(\d+(?:\.\d+)?)\s*(ms|s|%|KB|MB|GB)", "lat=123ms size=1.5GB cpu=42%")
```

---

## 4) Multiline entries and record boundaries
**What**: Stack traces and wrapped lines need grouping before parsing.
```python
from itertools import groupby

def records(lines):
    # new record starts when line begins with YYYY-MM-DD ...
    import re
    start = re.compile(r"^\d{4}-\d{2}-\d{2} ")
    buf = []
    for line in lines:
        if start.match(line) and buf:
            yield "".join(buf); buf = []
        buf.append(line)
    if buf:
        yield "".join(buf)

# then parse each record with a regex for the header and capture optional body
```
Alternative: use `re.S` (DOTALL) to match across newlines with explicit delimiters.

---

## 5) Greedy vs lazy and backtracking
**What**: Control how much text `.*` consumes.
```python
import re
s = "<tag>alpha</tag><tag>beta</tag>"
print(re.findall(r"<tag>(.*)</tag>", s))       # greedy → ['alpha</tag><tag>beta']
print(re.findall(r"<tag>(.*?)</tag>", s))      # lazy → ['alpha', 'beta']
```
Tips:
- Prefer explicit character classes over `.*?` when possible (e.g., `[^<]+`).  
- Anchor with `^` and `$` for line‑start/line‑end to reduce backtracking.

---

## 6) Named groups and data classes
**What**: Map regex groups to structured Python objects.
```python
import re
from dataclasses import dataclass

@dataclass
class Log:
    ts: str
    level: str
    user: str

pat = re.compile(r"^(?P<ts>\S+)\s+(?P<level>INFO|WARN|ERROR)\s+user=(?P<user>\w+)")
line = "2025-10-06T12:34:56Z INFO user=ned"
if m := pat.search(line):
    log = Log(**m.groupdict())
    print(log)
```

---

## 7) Streaming large files safely
**What**: Avoid loading entire files into memory.
```python
from pathlib import Path
import re

pat = re.compile(r"ERROR|CRITICAL|Traceback", re.I)
count = 0
with Path("/var/log/app.log").open("r", encoding="utf-8", errors="ignore") as f:
    for line in f:
        if pat.search(line):
            count += 1
print("errors:", count)
```
Tips:
- Use `errors="ignore"` or `replace` for dirty logs.  
- For very large files, consider `mmap` or chunked reading.  
- If multiple files rotate, iterate over `glob('app.log*')` and sort.

---

## 8) Extracting embedded JSON/YAML/TOML
**What**: Logs often contain JSON lines or config blocks; parse with real parsers.

### 8.1 JSON line per log
```python
import json
line = '{"ts":"2025-10-06","msg":"ok","lat":0.12}'
obj = json.loads(line)
print(obj["lat"])
```

### 8.2 JSON inside text
```python
import json, re
text = "payload={\"a\":1,\"b\":[2,3]} end"
if m := re.search(r"payload=(\{.*\})", text):
    obj = json.loads(m.group(1))
```
Be careful: `.*` breaks on nested braces; prefer a stack parser or locate matching braces.

### 8.3 YAML/TOML blocks
```python
from ruamel.yaml import YAML
import tomli

yaml = YAML()
with open("conf.yaml", "r", encoding="utf-8") as f:
    cfg = yaml.load(f)

with open("conf.toml", "rb") as f:
    cfg2 = tomli.load(f)
```

---

## 9) Timestamps, timezones, and sorting
**What**: Normalize times to compare and sort.
```python
from dateutil import parser, tz
s = "2025-10-06 12:34:56 +10:00"
dt = parser.parse(s).astimezone(tz.UTC)
print(dt.isoformat())
```
Guidelines:
- Parse once at the edge, store as aware UTC.  
- Include original string for audits.  
- For fixed formats, prefer `datetime.strptime` for speed.

---

## 10) Encodings and Unicode
**What**: Read text robustly across systems.
```python
from pathlib import Path
import chardet
raw = Path("mystery.log").read_bytes()
enc = chardet.detect(raw)["encoding"] or "utf-8"
text = raw.decode(enc, errors="replace")
```
Notes:
- Always write files with `encoding="utf-8"`.  
- Normalize whitespace with `splitlines()` and `strip()`.

---

## 11) Validation and error handling
**What**: Fail safely and surface bad inputs.
```python
import re
pat = re.compile(r"^user=(?P<user>\w+)$")
line = "user=ned extra"
if not (m := pat.fullmatch(line)):
    raise ValueError(f"bad line: {line!r}")
```
Prefer `fullmatch` over `search` when the whole line must match.

---

## 12) Troubleshooting
- **Regex matches too much**: replace `.*` with explicit classes (e.g., `[^\s]+`) and anchor with `^…$`.  
- **Slow patterns**: catastrophic backtracking; simplify groups, use lazy quantifiers, or add anchors.  
- **Multiline confusion**: set `re.M` for `^$` and `re.S` for dot‑matches‑newline.  
- **Encoding errors**: open with `encoding` and `errors`, or decode bytes after detection.  
- **JSON in logs fails**: logs often escape quotes; unescape before `json.loads` if needed.

---

## 13) Recap
```plaintext
Start with string methods → switch to regex for variability → use named groups and data classes → handle multiline records → stream large files → parse embedded JSON/YAML/TOML with real parsers → normalize timestamps and encodings → validate strictly → profile and tighten patterns
```

**Next**: Feed parsed data into CSV/Excel (`013_CSVEXCEL.md`), visualize (`011_GRAPHS.md`), or send to monitoring/alerts (`025_MONITORING_SCRIPTS.md`).