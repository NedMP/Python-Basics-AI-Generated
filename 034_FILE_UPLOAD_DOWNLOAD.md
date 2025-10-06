

# File Uploads & Downloads (HTTP, SFTP, S3, OneDrive, resumable) — updated Oct 6, 2025

Practical guide to moving files **reliably** with Python. Covers HTTP(S) uploads/downloads, **resumable** transfers, SFTP via Paramiko, Amazon **S3** with multipart, and **OneDrive** via Microsoft Graph upload sessions. Includes retries, integrity checks, progress bars, and `.env`-based config. Pair with `002_SETUP.md` for venv and `033_KEYCHAIN_SECRETS.md` for secret storage.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Secrets provided via `.env` or Keychain.  
- Large files may need chunking and resume support.

**Install**
```sh
pip install requests python-dotenv tqdm paramiko boto3 botocore msal
```

---

## Table of Contents
- [0) Safety and integrity](#0-safety-and-integrity)
- [1) HTTP downloads with resume](#1-http-downloads-with-resume)
- [2) HTTP uploads (multipart/form-data and PUT)](#2-http-uploads-multipartform-data-and-put)
- [3) SFTP upload/download with Paramiko](#3-sftp-uploaddownload-with-paramiko)
- [4) Amazon S3 uploads/downloads (multipart)](#4-amazon-s3-uploadsdownloads-multipart)
- [5) OneDrive via Microsoft Graph (upload sessions)](#5-onedrive-via-microsoft-graph-upload-sessions)
- [6) Progress bars and speed reporting](#6-progress-bars-and-speed-reporting)
- [7) Retries, backoff, and timeouts](#7-retries-backoff-and-timeouts)
- [8) Directory mirroring patterns](#8-directory-mirroring-patterns)
- [9) Troubleshooting](#9-troubleshooting)
- [10) Recap](#10-recap)

---

## 0) Safety and integrity
**What**: Protect credentials, verify content, and avoid partial overwrites.  
**Why**: Transfers fail; integrity checks and atomic moves prevent corruption.
- Keep secrets in `.env` or Keychain; never commit tokens.  
- Download to a temp file then **atomic rename**.  
- Verify content with **SHA‑256** or server‑returned checksums.

```python
# file: integrity.py
from pathlib import Path
import hashlib

BUF = 1024*1024

def sha256sum(p: Path) -> str:
    h = hashlib.sha256()
    with p.open('rb') as f:
        while chunk := f.read(BUF):
            h.update(chunk)
    return h.hexdigest()
```

---

## 1) HTTP downloads with resume
**What**: Stream to disk and resume using HTTP `Range` requests.  
**Why**: Large downloads survive network drops.
```python
# file: http_download_resume.py
import os, requests
from pathlib import Path

URL = os.getenv('FILE_URL', 'https://example.com/big.iso')
DEST = Path('downloads/big.iso'); DEST.parent.mkdir(exist_ok=True)
TEMP = DEST.with_suffix('.part')

pos = TEMP.stat().st_size if TEMP.exists() else 0
headers = {'Range': f'bytes={pos}-'} if pos else {}
with requests.get(URL, headers=headers, stream=True, timeout=30) as r:
    r.raise_for_status()
    mode = 'ab' if pos else 'wb'
    with TEMP.open(mode) as f:
        for chunk in r.iter_content(chunk_size=1024*1024):
            if chunk:
                f.write(chunk)
TEMP.replace(DEST)
print('saved', DEST)
```
Notes:
- Server must support `Accept-Ranges`. If not, restart from scratch.  
- Validate with `sha256sum(DEST)` if you have a reference hash.

---

## 2) HTTP uploads (multipart/form-data and PUT)
**What**: Send files to HTTP endpoints.  
**Why**: Many APIs accept uploads as multipart or raw PUT.
```python
# multipart/form-data
import requests
with open('report.csv','rb') as f:
    files = {'file': ('report.csv', f, 'text/csv')}
    r = requests.post('https://example.com/upload', files=files, timeout=30)
    r.raise_for_status()

# raw PUT
with open('video.mp4','rb') as f:
    r = requests.put('https://example.com/put/video.mp4', data=f, timeout=120)
    r.raise_for_status()
```
Tips:
- Send auth headers (`Authorization: Bearer ...`).  
- For very large uploads, use API‑provided **upload sessions** (see OneDrive) or S3 multipart.

---

## 3) SFTP upload/download with Paramiko
**What**: SSH‑based file transfer to servers.  
**Why**: Encrypted, widely available on Linux/BSD/macOS.
```python
# file: sftp_demo.py
from dotenv import load_dotenv; load_dotenv()
import os, paramiko
from pathlib import Path

HOST = os.getenv('SFTP_HOST')
USER = os.getenv('SFTP_USER')
KEY  = os.getenv('SFTP_KEY', str(Path.home()/'.ssh/id_ed25519'))

key = paramiko.Ed25519Key.from_private_key_file(KEY)
transport = paramiko.Transport((HOST, 22))
transport.connect(username=USER, pkey=key)
sftp = paramiko.SFTPClient.from_transport(transport)

# upload
sftp.put('local/file.txt', '/remote/dir/file.txt')
# download
sftp.get('/remote/dir/file.txt', 'downloads/file.txt')

sftp.close(); transport.close()
```
Notes:
- Use keys, not passwords.  
- For progress, supply a callback to `put`/`get`.

---

## 4) Amazon S3 uploads/downloads (multipart)
**What**: Use boto3 high‑level transfers with automatic multipart and retries.  
**Why**: Efficient for large files and unstable links.
```python
# file: s3_transfer.py
import boto3
from boto3.s3.transfer import TransferConfig

bucket = 'my-bucket'; key = 'uploads/big.bin'
MB = 1024 * 1024
cfg = TransferConfig(multipart_threshold=8*MB, multipart_chunksize=8*MB, max_concurrency=8)

s3 = boto3.client('s3', region_name='ap-southeast-2')
# upload
s3.upload_file('big.bin', bucket, key, Config=cfg, ExtraArgs={'ServerSideEncryption':'AES256'})
# download
s3.download_file(bucket, key, 'downloads/big.bin', Config=cfg)
```
Tips:
- `upload_file` and `download_file` include retries by default.  
- For checksums, enable `ChecksumAlgorithm='SHA256'` when supported and compare.

---

## 5) OneDrive via Microsoft Graph (upload sessions)
**What**: Resumable chunked uploads using Graph **upload sessions**.  
**Why**: Reliable for large files and flaky networks.

### 5.1 Acquire token with MSAL (device code flow for personal/testing)
```python
# file: msal_token.py
import os, msal
CLIENT_ID = os.getenv('AZURE_CLIENT_ID')
TENANT = os.getenv('AZURE_TENANT_ID', 'common')
app = msal.PublicClientApplication(CLIENT_ID, authority=f"https://login.microsoftonline.com/{TENANT}")
scopes = ["Files.ReadWrite.All", "offline_access"]
flow = app.initiate_device_flow(scopes=scopes)
print(flow['message'])  # visit URL and enter code
result = app.acquire_token_by_device_flow(flow)
print(result['access_token'][:20], '...')
```

### 5.2 Create upload session and send chunks
```python
# file: onedrive_upload.py
import os, json, requests
from pathlib import Path

TOKEN = os.getenv('GRAPH_TOKEN')  # set from msal_token result
FILE  = Path('big.bin')
NAME  = FILE.name
# OneDrive root path example; adjust for SharePoint/site drives
create = requests.post(
  f"https://graph.microsoft.com/v1.0/me/drive/root:/{NAME}:/createUploadSession",
  headers={'Authorization': f'Bearer {TOKEN}'},
  json={"item": {"@microsoft.graph.conflictBehavior": "replace"}}, timeout=30)
create.raise_for_status(); uploadUrl = create.json()['uploadUrl']

CHUNK = 5 * 1024 * 1024
size = FILE.stat().st_size
with FILE.open('rb') as f:
    start = 0
    while True:
        data = f.read(CHUNK)
        if not data: break
        end = start + len(data) - 1
        headers = {
          'Authorization': f'Bearer {TOKEN}',
          'Content-Length': str(len(data)),
          'Content-Range': f'bytes {start}-{end}/{size}'
        }
        r = requests.put(uploadUrl, headers=headers, data=data, timeout=120)
        if r.status_code not in (200, 201, 202): r.raise_for_status()
        start = end + 1
print('uploaded', NAME)
```
Notes:
- Upload sessions can be **resumed** by reusing `uploadUrl` until it expires.  
- For enterprise, prefer **client credentials** and delegated permissions as per policy.

---

## 6) Progress bars and speed reporting
**What**: Display transfer progress for UX and debugging.  
**Why**: Visibility on large files.
```python
# file: http_download_tqdm.py
import requests
from tqdm import tqdm
from pathlib import Path

url = 'https://speed.hetzner.de/100MB.bin'
dest = Path('downloads/100MB.bin'); dest.parent.mkdir(exist_ok=True)
with requests.get(url, stream=True, timeout=30) as r:
    r.raise_for_status()
    total = int(r.headers.get('Content-Length', 0))
    with tqdm(total=total, unit='B', unit_scale=True, desc=dest.name) as bar:
        with dest.open('wb') as f:
            for chunk in r.iter_content(1024*1024):
                if chunk:
                    f.write(chunk); bar.update(len(chunk))
```

---

## 7) Retries, backoff, and timeouts
**What**: Stabilize network calls.  
**Why**: Transient failures are common; do not crash on first error.
```python
# file: retry.py
import time, random, requests

def post_with_retry(url, *, json=None, data=None, headers=None, attempts=5, base=0.5, timeout=30):
    for i in range(attempts):
        r = requests.post(url, json=json, data=data, headers=headers, timeout=timeout)
        if r.status_code in (200, 201, 202, 204):
            return r
        if r.status_code == 429:
            wait = float(r.headers.get('Retry-After', base*(2**i)))
            time.sleep(wait); continue
        if 500 <= r.status_code < 600:
            time.sleep(base*(2**i) + random.random()/10); continue
        r.raise_for_status()
    r.raise_for_status()
```
Always set **timeouts** for every request.

---

## 8) Directory mirroring patterns
**What**: Keep a local and remote folder in sync at the file level.  
**Why**: Incremental updates are faster and cheaper than re‑uploads.
- S3: use `upload_file` with consistent keys and prune with `list_objects_v2` + deletes.  
- SFTP: compare file sizes and mtimes; upload changed files only.  
- HTTP servers: prefer an API with object listing and ETags for change detection.

Skeleton for SFTP incremental:
```python
# compare local Path.stat().st_mtime_ns with sftp.stat(remote).st_mtime
```
For robust mirroring consider `rsync` (see `030_BACKUP_SCRIPTS.md`).

---

## 9) Troubleshooting
- **HTTP 416 or restarts on resume**: server lacks Range support; remove resume headers.  
- **SFTP `Permission denied`**: path or ownership wrong; verify key and server auth.  
- **S3 slow**: increase `max_concurrency`, tune chunk size, use VPC endpoints, verify region.  
- **Graph 401/403**: token expired or missing scope; re-auth and ensure `Files.ReadWrite.All`.  
- **Checksum mismatch**: re-download; verify no proxy corruption.  
- **Unicode paths**: always use `Path` and `encoding='utf-8'` when reading metadata.

---

## 10) Recap
```plaintext
Set secrets → stream with timeouts → resume where possible → verify hashes → use multipart/sessions for large files → show progress → add retries/backoff → mirror incrementally when needed
```

**Next**: Schedule transfers with `026_TASK_SCHEDULING.md`, alert results via `031_NOTIFICATION_AUTOMATION.md`, and verify contents using `030_BACKUP_SCRIPTS.md` integrity patterns.