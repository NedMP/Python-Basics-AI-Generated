

# Backup Scripts with Python — updated Oct 6, 2025

Practical guide to automating **backups** with Python: creating **archives** (tar/zip), doing **incremental syncs**, maintaining **retention**, and verifying **data integrity** with hashes. Includes optional encryption and cloud off‑site patterns. Pair with `026_TASK_SCHEDULING.md` for running on a schedule and `016_LOGGING.md` for structured logs.

---

**Assumptions**  
- macOS (zsh) or Linux, Python 3.12–3.13 in a venv.  
- Source folders are readable and targets have enough space.  
- You prefer **copy‑pasteable** scripts that are easy to adapt.

**Install**
```sh
pip install python-dotenv rich boto3 azure-storage-blob
```
*(Cloud libs optional; only install what you use.)*

---

## Table of Contents
- [0) Backup strategy and safety](#0-backup-strategy-and-safety)
- [1) Directory snapshot → tar.gz archive](#1-directory-snapshot--targz-archive)
- [2) Zip archives with include/exclude](#2-zip-archives-with-includeexclude)
- [3) Incremental sync with rsync (preferred)](#3-incremental-sync-with-rsync-preferred)
- [4) Pure-Python incremental copy with hashing](#4-pure-python-incremental-copy-with-hashing)
- [5) Verify integrity with checksums](#5-verify-integrity-with-checksums)
- [6) Retention: rotate and prune old backups](#6-retention-rotate-and-prune-old-backups)
- [7) Encryption at rest and in transit](#7-encryption-at-rest-and-in-transit)
- [8) Off‑site copies: S3/Azure Blob](#8-off-site-copies-s3azure-blob)
- [9) Restore tests and drills](#9-restore-tests-and-drills)
- [10) Scheduling](#10-scheduling)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Backup strategy and safety
**What**: Define goals before writing code.  
**Why**: Backups that can’t be restored are useless.
- Follow **3‑2‑1**: 3 copies, 2 media, 1 off‑site.  
- Separate **Cold** (archives) vs **Hot** (incremental syncs).  
- Log every run, capture file counts, sizes, duration, and errors.  
- Never overwrite your only copy; stage to `work/` then atomically move.

---

## 1) Directory snapshot → tar.gz archive
**What**: Create compressed point‑in‑time snapshots.  
**Why**: Simple and portable.
```python
# file: make_tar.py
from pathlib import Path
import tarfile, time

SRC = Path("/Users/ned/Documents/projects")
DEST = Path("/Backups/archives"); DEST.mkdir(parents=True, exist_ok=True)
stamp = time.strftime("%Y%m%d-%H%M%S")
arc = DEST / f"projects_{stamp}.tar.gz"

with tarfile.open(arc, "w:gz", compresslevel=6) as tar:
    tar.add(SRC, arcname=SRC.name)
print("wrote", arc)
```
Notes:
- Excludes: pre‑filter with a file walker (see Section 2) or supply a `filter` callable.

---

## 2) Zip archives with include/exclude
**What**: Finer control over entries and patterns.
```python
# file: make_zip.py
from pathlib import Path
from zipfile import ZipFile, ZIP_DEFLATED
import fnmatch, time

SRC = Path("/Users/ned/Documents/projects")
DEST = Path("/Backups/archives"); DEST.mkdir(parents=True, exist_ok=True)
INCLUDE = ["**/*.py", "**/*.md", "**/*.txt"]
EXCLUDE = ["**/__pycache__/**", "**/.venv/**", "**/.git/**"]

stamp = time.strftime("%Y%m%d-%H%M%S")
zip_path = DEST / f"projects_{stamp}.zip"

with ZipFile(zip_path, "w", compression=ZIP_DEFLATED, compresslevel=6) as z:
    for path in SRC.rglob("**/*"):
        rel = path.relative_to(SRC)
        s = rel.as_posix()
        if any(fnmatch.fnmatch(s, e) for e in EXCLUDE):
            continue
        if not any(fnmatch.fnmatch(s, i) for i in INCLUDE):
            continue
        if path.is_file():
            z.write(path, arcname=rel)
print("wrote", zip_path)
```
Tip: Keep includes explicit to avoid archiving junk.

---

## 3) Incremental sync with rsync (preferred)
**What**: Copy only changed files, preserve metadata, and maintain a mirror.  
**Why**: Fast, reliable, proven.
```python
# file: sync_rsync.py
import subprocess
SRC = "/Users/ned/Documents/projects/"
DST = "/Volumes/BackupDrive/projects/"
args = [
    "rsync","-aHAX","--delete","--numeric-ids",
    "--exclude",".venv/","--exclude","__pycache__/","--exclude",".git/",
    SRC, DST
]
subprocess.run(args, check=True)
```
Notes:
- Trailing slash on SRC copies **contents** of folder. Remove it to copy folder itself.  
- `--delete` matches deletions; omit if you prefer only additive sync.

---

## 4) Pure-Python incremental copy with hashing
**What**: When `rsync` isn’t available.  
**Why**: Control and portability at the cost of speed.
```python
# file: sync_hash.py
from pathlib import Path
import shutil, hashlib

SRC = Path("/Users/ned/Documents/projects")
DST = Path("/Backups/mirror"); DST.mkdir(parents=True, exist_ok=True)

BUF = 1024*1024

def sha256(p: Path) -> str:
    h = hashlib.sha256()
    with p.open('rb') as f:
        while chunk := f.read(BUF):
            h.update(chunk)
    return h.hexdigest()

for src in SRC.rglob('*'):
    if src.is_dir():
        (DST/src.relative_to(SRC)).mkdir(exist_ok=True)
        continue
    rel = src.relative_to(SRC)
    dst = DST/rel
    if not dst.exists() or sha256(src) != sha256(dst):
        dst.parent.mkdir(parents=True, exist_ok=True)
        shutil.copy2(src, dst)
```
Caveat: Hashing is CPU‑intensive; prefer size+mtime checks first if performance matters.

---

## 5) Verify integrity with checksums
**What**: Detect silent corruption in transit or at rest.  
**Why**: Assurance that backups match sources.
```python
# file: verify_tree.py
from pathlib import Path
import hashlib, json

SRC = Path("/Users/ned/Documents/projects")
DST = Path("/Backups/mirror")

BUF = 1024*1024

def file_hash(p: Path) -> str:
    h = hashlib.sha256()
    with p.open('rb') as f:
        while chunk := f.read(BUF):
            h.update(chunk)
    return h.hexdigest()

manifest = {}
for s in SRC.rglob('*'):
    if s.is_file():
        manifest[str(s.relative_to(SRC))] = file_hash(s)

bad = []
for rel, h in manifest.items():
    d = DST/rel
    if not d.exists() or file_hash(d) != h:
        bad.append(rel)

print("mismatches:", bad)
Path("checksums.json").write_text(json.dumps(manifest, indent=2))
```
Store manifests with the backup for later verification.

---

## 6) Retention: rotate and prune old backups
**What**: Keep recent snapshots and prune old ones.  
**Why**: Control storage while retaining useful restore points.
```python
# file: prune.py
from pathlib import Path
import re
from datetime import datetime, timedelta

DEST = Path("/Backups/archives")
KEEP_DAYS = 14
pat = re.compile(r"projects_(\d{8}-\d{6})\.(?:tar\.gz|zip)$")

cut = datetime.now() - timedelta(days=KEEP_DAYS)
for p in DEST.iterdir():
    m = pat.match(p.name)
    if not m:
        continue
    ts = datetime.strptime(m.group(1), "%Y%m%d-%H%M%S")
    if ts < cut:
        p.unlink()
        print("deleted", p)
```
For calendar rotation (daily/weekly/monthly), maintain folders like `daily/`, `weekly/`, `monthly/`.

---

## 7) Encryption at rest and in transit
**What**: Protect sensitive data.  
**Why**: Backups often contain credentials or PII.

### 7.1 Zip with a password (basic)
```python
# built-in zipfile has no strong encryption. Prefer external tools.
```

### 7.2 GPG symmetric encryption (strong)
```sh
gpg --symmetric --cipher-algo AES256 projects_20251006-120000.tar.gz
```
Automate via `subprocess.run(["gpg", "--symmetric", "--cipher-algo", "AES256", str(arc)])`.

### 7.3 Encrypt before cloud upload
- Generate keys and restrict access.  
- Or rely on server‑side encryption (S3 SSE‑S3/SSE‑KMS, Azure SSE) with tight IAM.

---

## 8) Off‑site copies: S3/Azure Blob
**What**: Store a copy off‑site for disaster recovery.

### 8.1 Amazon S3
```python
import boto3
s3 = boto3.client("s3")
s3.upload_file("/Backups/archives/projects_20251006-120000.tar.gz", "my-bucket", "archives/projects_20251006-120000.tar.gz")
```
Use `ExtraArgs={"ServerSideEncryption":"AES256"}` or KMS where required.

### 8.2 Azure Blob Storage
```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient
import os

account = os.environ["AZURE_STORAGE_ACCOUNT"]
cred = DefaultAzureCredential()
svc = BlobServiceClient(f"https://{account}.blob.core.windows.net", credential=cred)
container = svc.get_container_client("archives"); container.create_container(exist_ok=True)
with open("/Backups/archives/projects_20251006-120000.tar.gz","rb") as f:
    container.upload_blob("projects_20251006-120000.tar.gz", f, overwrite=True)
```
Grant `Storage Blob Data Contributor` to your principal.

---

## 9) Restore tests and drills
**What**: Prove you can recover.  
**Why**: Detects missing permissions, corrupt archives, or wrong paths.
- Periodically extract a random archive to a temp folder and compare checksums.  
- For mirrors, pick sample files and byte‑compare with source.  
- Script a **full restore** to a sandbox path at least monthly.

---

## 10) Scheduling
**What**: Run regularly without manual steps.  
**Why**: Fresh backups are the only backups.
- Desktop: `schedule` loop or `launchd` (macOS).  
- Server: cron, systemd timers, or APScheduler.  
- See `026_TASK_SCHEDULING.md` for examples with lock files and retries.

---

## 11) Troubleshooting
- **Slow backups**: exclude virtualenvs, caches, and build artifacts; use rsync; increase compression level thoughtfully.  
- **Permission denied**: run with proper rights; use `sudo` where appropriate; preserve ownership with rsync flags.  
- **Changed files during backup**: snapshot to a staging area first or use filesystem snapshots if available.  
- **Corrupt archives**: verify after write; avoid network filesystems for active archives.  
- **Clock drift**: filenames rely on correct time; sync NTP.

---

## 12) Recap
```plaintext
Plan (3‑2‑1) → archive or sync → verify checksums → rotate → encrypt → off‑site → schedule → test restores
```

**Next**: Pipe results to notifications (`031_NOTIFICATION_AUTOMATION.md`), store manifests alongside archives, and integrate with `029_DATA_CLEANUP.md` for pre‑backup sanitization if needed.