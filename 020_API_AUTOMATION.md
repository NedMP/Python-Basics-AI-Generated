

# Automating REST APIs with Python — updated Oct 6, 2025

Practical guide for scripting against HTTP/REST APIs. Covers authentication patterns, sessions, timeouts, JSON handling, CRUD operations, pagination styles, retries and rate limits, uploads/downloads, idempotency, async with `httpx`, basic validation, and troubleshooting. Use with `002_SETUP.md` for environment and `.env` handling.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Keep secrets in `.env` and load with `python-dotenv`.

---

## Table of Contents
- [0) Install and project scaffold](#0-install-and-project-scaffold)
- [1) Requests basics: sessions, timeouts, JSON](#1-requests-basics-sessions-timeouts-json)
- [2) Authentication patterns](#2-authentication-patterns)
- [3) CRUD examples](#3-crud-examples)
- [4) Pagination patterns](#4-pagination-patterns)
- [5) Retries, rate limits, and backoff](#5-retries-rate-limits-and-backoff)
- [6) File upload and download](#6-file-upload-and-download)
- [7) Idempotency and safety](#7-idempotency-and-safety)
- [8) Async requests with httpx](#8-async-requests-with-httpx)
- [9) Validating responses](#9-validating-responses)
- [10) Error handling and diagnostics](#10-error-handling-and-diagnostics)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Install and project scaffold
**What**: Set up dependencies and a small structure you can reuse.
```sh
pip install requests python-dotenv httpx "urllib3>=2" pydantic
mkdir -p api-demo && cd api-demo && touch api.py .env
```
`.env` template:
```
API_BASE=https://api.example.com
API_KEY=your_key_here
TOKEN=your_bearer_token
```

---

## 1) Requests basics: sessions, timeouts, JSON
**What**: Use a `Session` for connection pooling and shared headers. Always set timeouts.
```python
# api.py
from __future__ import annotations
import os
from dotenv import load_dotenv
import requests

load_dotenv()
BASE = os.getenv("API_BASE", "https://httpbin.org")

session = requests.Session()
session.headers.update({
    "User-Agent": "python-notes/1.0",
    "Accept": "application/json",
})

# GET with timeout and JSON decoding
r = session.get(f"{BASE}/get", timeout=10)
r.raise_for_status()
data = r.json()
print(data)
```
Notes:
- `timeout` is mandatory. Use a number or tuple `(connect, read)`.
- `raise_for_status()` converts HTTP 4xx/5xx into exceptions.

---

## 2) Authentication patterns
**What**: Common auth schemes you will meet.

### 2.1 API key in header
```python
session.headers["X-API-Key"] = os.getenv("API_KEY", "")
```

### 2.2 Bearer token
```python
session.headers["Authorization"] = f"Bearer {os.getenv('TOKEN','')}"
```

### 2.3 Basic auth
```python
from requests.auth import HTTPBasicAuth
r = session.get(f"{BASE}/basic-auth/user/pass", auth=HTTPBasicAuth("user", "pass"), timeout=10)
```

### 2.4 OAuth2 client credentials (quick pattern)
For real OAuth, prefer a dedicated library (`authlib`, `msal`). Minimal example with a token endpoint:
```python
import requests
resp = requests.post("https://auth.example.com/oauth2/token", data={
    "grant_type": "client_credentials",
    "client_id": os.getenv("CLIENT_ID"),
    "client_secret": os.getenv("CLIENT_SECRET"),
    "scope": "api.read api.write",
}, timeout=15)
resp.raise_for_status()
access_token = resp.json()["access_token"]
session.headers["Authorization"] = f"Bearer {access_token}"
```

---

## 3) CRUD examples
**What**: Create, read, update, delete with JSON.
```python
# Create
payload = {"name": "widget", "price": 9.99}
r = session.post(f"{BASE}/items", json=payload, timeout=15)
r.raise_for_status()
item = r.json()
item_id = item.get("id")

# Read
r = session.get(f"{BASE}/items/{item_id}", timeout=10)
r.raise_for_status(); item = r.json()

# Update (PUT or PATCH)
updates = {"price": 12.50}
r = session.patch(f"{BASE}/items/{item_id}", json=updates, timeout=15)
r.raise_for_status(); item = r.json()

# Delete
r = session.delete(f"{BASE}/items/{item_id}", timeout=10)
if r.status_code not in (200, 202, 204):
    r.raise_for_status()
```
Tips:
- Prefer `json=` over `data=` for JSON bodies.  
- Expect `201 Created` on create; location may be in `r.headers['Location']`.

---

## 4) Pagination patterns
**What**: Iterate all results safely. Different APIs, different shapes.

### 4.1 Page + size
```python
def iter_pages(url: str, size=100):
    page = 1
    while True:
        r = session.get(url, params={"page": page, "per_page": size}, timeout=20)
        r.raise_for_status()
        items = r.json()
        if not items:
            break
        for it in items:
            yield it
        page += 1
```

### 4.2 Offset + limit
```python
def iter_offset(url: str, limit=100):
    offset = 0
    while True:
        r = session.get(url, params={"offset": offset, "limit": limit}, timeout=20)
        r.raise_for_status()
        data = r.json()
        items = data.get("items", [])
        if not items:
            break
        for it in items:
            yield it
        offset += len(items)
```

### 4.3 Cursor/next link
```python
def iter_cursor(url: str):
    next_url = url
    while next_url:
        r = session.get(next_url, timeout=20)
        r.raise_for_status()
        data = r.json()
        for it in data.get("items", []):
            yield it
        next_url = data.get("next") or r.links.get("next", {}).get("url")
```

---

## 5) Retries, rate limits, and backoff
**What**: Handle transient failures and 429/5xx politely.
```python
import requests
from urllib3.util.retry import Retry
from requests.adapters import HTTPAdapter

retry = Retry(
    total=5,
    backoff_factor=0.5,
    status_forcelist=(429, 500, 502, 503, 504),
    allowed_methods=("GET","POST","PUT","PATCH","DELETE"),
    raise_on_status=False,
)
adapter = HTTPAdapter(max_retries=retry)
session.mount("http://", adapter)
session.mount("https://", adapter)
```
Honor server `Retry-After` header automatically (urllib3 does). For 429 with per‑user quotas, consider queueing and exponential backoff with jitter.

---

## 6) File upload and download
**Upload (multipart/form-data)**
```python
files = {"file": ("report.csv", open("report.csv","rb"), "text/csv")}
extra = {"project": (None, "alpha")}
r = session.post(f"{BASE}/upload", files=files, data=extra, timeout=60)
r.raise_for_status()
```
**Download streaming**
```python
with session.get(f"{BASE}/bigfile", stream=True, timeout=(5, 120)) as r:
    r.raise_for_status()
    with open("bigfile.bin", "wb") as f:
        for chunk in r.iter_content(chunk_size=1024*1024):
            if chunk:
                f.write(chunk)
```

---

## 7) Idempotency and safety
**What**: Prevent duplicate side‑effects on retries.
- Use **Idempotency‑Key** headers for POST if the API supports it.
```python
import uuid
session.headers["Idempotency-Key"] = str(uuid.uuid4())
```
- For destructive ops, implement **confirm flags** and **dry runs** in your scripts.  
- Log request IDs from response headers for audits (e.g., `X-Request-ID`).

---

## 8) Async requests with httpx
**What**: Concurrency for many API calls; backpressure with semaphores.
```python
# file: async_bulk.py
import asyncio, os
import httpx
from dotenv import load_dotenv
load_dotenv()
BASE = os.getenv("API_BASE", "https://httpbin.org")

async def fetch(client: httpx.AsyncClient, i: int):
    r = await client.get(f"{BASE}/get", timeout=10)
    r.raise_for_status()
    return r.json()

async def main():
    limits = httpx.Limits(max_connections=20)
    async with httpx.AsyncClient(limits=limits, headers={"Accept":"application/json"}) as client:
        sem = asyncio.Semaphore(10)
        async def guarded(i):
            async with sem:
                return await fetch(client, i)
        results = await asyncio.gather(*(guarded(i) for i in range(100)))
        print(len(results))

asyncio.run(main())
```
Notes:
- Tune `Limits` and semaphore to respect provider quotas.  
- `httpx` supports retries via `transport` wrappers or manual retry logic.

---

## 9) Validating responses
**What**: Fail early when schemas change.
```python
from pydantic import BaseModel

class Item(BaseModel):
    id: int
    name: str
    price: float

r = session.get(f"{BASE}/items/123", timeout=10)
r.raise_for_status()
item = Item.model_validate(r.json())
print(item)
```
Use minimal models for critical fields rather than mirroring entire payloads.

---

## 10) Error handling and diagnostics
**What**: Standardize exceptions and logs.
```python
import logging
log = logging.getLogger("api")

try:
    r = session.get(f"{BASE}/status/503", timeout=5)
    r.raise_for_status()
except requests.HTTPError as e:
    log.error("http %s: %s", e.response.status_code if e.response else "?", e)
except requests.Timeout:
    log.error("timeout")
except requests.RequestException as e:
    log.error("network error: %s", e)
```
Attach context: endpoint, method, correlation IDs, attempt count, and elapsed times.

---

## 11) Troubleshooting
- **`SSL: CERTIFICATE_VERIFY_FAILED`**: update `certifi`; avoid `verify=False`.  
- **Hanging requests**: forgot `timeout`; set both connect and read.  
- **Retries not firing**: ensure you mounted an `HTTPAdapter` with `Retry` on the same session.  
- **Pagination stops early**: provider may cap page size; follow `next` links rather than counting.  
- **429s**: back off and honor `Retry-After`; reduce concurrency.  
- **Unicode issues**: rely on `response.json()` or specify `response.encoding` before `response.text`.

---

## 12) Recap
```plaintext
Create Session → add auth headers → use timeouts → perform CRUD → iterate pagination → configure retries/backoff → stream files → add idempotency keys → async when needed → validate critical fields → log and handle errors
```

**Next**: For CLI wrapping and packaging, see the planned `035_SCRIPT_PACKAGING.md`. For scheduled API pulls or pushes, integrate with `026_TASK_SCHEDULING.md`. For feeding results into reports, see `013_CSVEXCEL.md` and email via `014_EMAILS.md`. 