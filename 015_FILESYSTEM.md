

# Python Filesystem Operations — updated Oct 5, 2025

Practical guide to reading, writing, moving, copying, deleting files; traversing directories; handling paths; metadata and permissions; temp files; and monitoring changes safely. Uses modern **pathlib** first, with `os`, `shutil`, and `watchdog` where appropriate. Use with `002_SETUP.md` for environment setup and `.env` handling.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 inside a venv.  
- Examples are copy‑pasteable; adjust paths to your workspace.

---

## Table of Contents
- [0) Why pathlib](#0-why-pathlib)
- [1) Paths and basic operations](#1-paths-and-basic-operations)
- [2) Read and write files (text/binary)](#2-read-and-write-files-textbinary)
- [3) Create, list, and traverse directories](#3-create-list-and-traverse-directories)
- [4) Copy, move, delete, and safe patterns](#4-copy-move-delete-and-safe-patterns)
- [5) Globbing and file selection](#5-globbing-and-file-selection)
- [6) File metadata, permissions, ownership](#6-file-metadata-permissions-ownership)
- [7) Large files and streaming I/O](#7-large-files-and-streaming-io)
- [8) Temporary files and working areas](#8-temporary-files-and-working-areas)
- [9) Watching files for changes](#9-watching-files-for-changes)
- [10) Archives: zip and tar](#10-archives-zip-and-tar)
- [11) Common pitfalls and troubleshooting](#11-common-pitfalls-and-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Why pathlib
**What**: `pathlib.Path` is an object‑oriented API for filesystem paths.  
**Why**: Cleaner, cross‑platform, fewer string bugs.
```python
from pathlib import Path
p = Path.home() / "Code_Stuff" / "files-demo"
print(p)
```

---

## 1) Paths and basic operations
**What**: Build and inspect paths, ensure directories exist, resolve symlinks.
```python
from pathlib import Path
root = Path.home() / "Code_Stuff" / "files-demo"
root.mkdir(parents=True, exist_ok=True)

file = root / "example.txt"
print(file.name, file.suffix, file.stem)
print(file.parent)
print(file.exists(), file.is_file(), file.is_dir())
print(file.resolve())       # absolute path
print(file.expanduser())
```
Notes:
- Prefer `/` operator to join paths.  
- `mkdir(parents=True, exist_ok=True)` is idempotent.

---

## 2) Read and write files (text/binary)
**What**: Safe text and binary I/O with explicit encodings and newline handling.
```python
from pathlib import Path
file = Path("example.txt")

# write text
file.write_text("Hello, filesystem!\n", encoding="utf-8")

# read text
content = file.read_text(encoding="utf-8")
print(content)

# append
with file.open("a", encoding="utf-8") as f:
    f.write("Another line\n")

# binary
binfile = Path("image.bin")
binfile.write_bytes(b"\x00\x01\x02")
data = binfile.read_bytes()
```
Tips:
- Always set `encoding="utf-8"` for text.  
- Use context managers for custom modes or buffering.

---

## 3) Create, list, and traverse directories
**What**: Make directories, list children, walk trees.
```python
from pathlib import Path
root = Path("demo")
(root / "inbox").mkdir(parents=True, exist_ok=True)
(root / "outbox").mkdir(parents=True, exist_ok=True)

# list entries (non-recursive)
for p in root.iterdir():
    print(p, "dir" if p.is_dir() else "file")

# recursive traversal
for p in root.rglob("*.txt"):
    print("TXT:", p)
```
For custom walks with stats or pruning, use `os.walk`.

---

## 4) Copy, move, delete, and safe patterns
**What**: Modify the filesystem with `shutil` and Path helpers.  
**Why**: Need idempotent, reversible actions and careful deletes.
```python
from pathlib import Path
import shutil
src = Path("demo/inbox/readme.txt")
dst = Path("demo/outbox/readme.txt")

# ensure source exists for demo
src.parent.mkdir(parents=True, exist_ok=True)
src.write_text("content\n", encoding="utf-8")

# copy
shutil.copy2(src, dst)         # preserves metadata where possible

# move/rename
dst_renamed = dst.with_stem(dst.stem + "_v2")
shutil.move(str(dst), dst_renamed)

# delete file safely
if dst_renamed.exists():
    dst_renamed.unlink()       # remove single file

# delete directory tree (use with care)
out = Path("demo/outbox")
if out.exists():
    shutil.rmtree(out)
```
Safe patterns:
- Prefer **move** into a `trash/` or backup dir before permanent delete.  
- Confirm targets with strict globs (avoid `rm -rf` equivalents on broad patterns).  
- Use `exists()` checks and `try/except` around destructive operations.

---

## 5) Globbing and file selection
**What**: Select files by patterns and extensions.
```python
from pathlib import Path
root = Path("demo")
for p in root.glob("**/*.log"):
    if p.stat().st_size > 0:
        print("log:", p)
```
Notes:
- `glob("*.ext")` is non‑recursive; `rglob("*.ext")` is recursive.  
- Prefer pathlib globs over manual string checks.

---

## 6) File metadata, permissions, ownership
**What**: Inspect and modify metadata.
```python
from pathlib import Path
import os, stat, time
p = Path("example.txt")
info = p.stat()
print("bytes", info.st_size)
print("modified", time.ctime(info.st_mtime))

# permissions (POSIX)
p.chmod(0o644)

# make executable
p.chmod(p.stat().st_mode | stat.S_IXUSR)

# ownership (UNIX only; requires privileges)
# os.chown(p, uid, gid)
```
Caution: On macOS, extended attributes and quarantine flags may affect behavior when downloading files.

---

## 7) Large files and streaming I/O
**What**: Process big files without loading into RAM.
```python
from pathlib import Path

lines = 0
with Path("huge.csv").open("r", encoding="utf-8", newline="") as f:
    for _ in f:
        lines += 1
print("lines:", lines)
```
Binary streaming with chunks:
```python
from pathlib import Path

src = Path("big.bin"); dst = Path("big.copy")
with src.open("rb") as fi, dst.open("wb") as fo:
    while chunk := fi.read(1024 * 1024):  # 1 MiB
        fo.write(chunk)
```
Tip: Prefer `newline=""` for CSV; see `013_CSVEXCEL.md` for parsing.

---

## 8) Temporary files and working areas
**What**: Use OS‑managed temp areas to avoid collisions and auto‑cleanup.
```python
import tempfile, shutil
from pathlib import Path

with tempfile.TemporaryDirectory() as td:
    tmpdir = Path(td)
    p = tmpdir / "work.txt"
    p.write_text("temp\n", encoding="utf-8")
    print(p.read_text(encoding="utf-8"))
# folder auto‑deleted here
```
For single temp files:
```python
import tempfile
with tempfile.NamedTemporaryFile(delete=False, suffix=".txt") as tf:
    tf.write(b"hello\n")
    print(tf.name)
```

---

## 9) Watching files for changes
**What**: React to filesystem events (create/modify/delete/rename).  
**Why**: Trigger processing pipelines, reload configs, or sync folders.
```sh
pip install watchdog
```
```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
from pathlib import Path
import time

class Handler(FileSystemEventHandler):
    def on_modified(self, event):
        if not event.is_directory:
            print("modified:", event.src_path)

watch = Observer()
root = Path("demo")
root.mkdir(exist_ok=True)
watch.schedule(Handler(), str(root), recursive=True)
watch.start()
try:
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    watch.stop()
watch.join()
```
Tips:
- Debounce rapid events if launching heavy work.  
- On macOS, large trees may require FSEvents tuning; keep watch scopes tight.

---

## 10) Archives: zip and tar
**What**: Package or extract files programmatically.
```python
from pathlib import Path
import shutil

root = Path("demo")
shutil.make_archive("demo_backup", "zip", root_dir=root)

# extract
shutil.unpack_archive("demo_backup.zip", "restore")
```
`zipfile`/`tarfile` offer finer control when needed (permissions, compression levels).

---

## 11) Common pitfalls and troubleshooting
- **Encoding errors**: specify `encoding="utf-8"` for text I/O.  
- **Permission denied**: check owner/permissions; avoid operating on files opened by other processes.  
- **Path confusion**: print `Path.cwd()` and prefer absolute paths or project‑relative bases.  
- **Overwriting data**: write to a temp file then `replace()` atomically.
```python
from pathlib import Path
src = Path("data.txt")
new = src.with_suffix(".tmp")
new.write_text("new content\n", encoding="utf-8")
new.replace(src)  # atomic on POSIX
```
- **Deleting wrong targets**: log targets, require confirmations in destructive tools, and use dry‑run flags.  
- **Long paths / strange chars**: normalize with `Path.resolve()`; avoid naive `str.replace` on paths.

---

## 12) Recap
```plaintext
Use pathlib → build paths → read/write text or binary → traverse and select files → copy/move/delete safely → inspect metadata and permissions → stream big files → use temp dirs → watch with watchdog → zip/tar when needed
```

**Next**: For CSV/Excel parsing see `013_CSVEXCEL.md`; for backup strategies and rotation see the planned `030_BACKUP_SCRIPTS.md`; for notifications on file changes see `031_NOTIFICATION_AUTOMATION.md`. 