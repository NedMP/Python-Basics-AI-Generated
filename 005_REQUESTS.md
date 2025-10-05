

# HTTP Requests in Python (requests + httpx) — updated Oct 5, 2025

Beginner‑friendly guide to making web requests in Python. Covers HTTP methods, headers, query strings, forms, JSON, files, timeouts, retries, sessions, auth, streaming, and common pitfalls. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13.  
- Run inside an activated virtual environment.  
- We use **`requests`** for synchronous code, and **`httpx`** (optional) for modern sync/async and HTTP/2.

---

## Table of Contents
- [0) Install and quick test](#0-install-and-quick-test)
- [1) HTTP basics: methods and URLs](#1-http-basics-methods-and-urls)
- [2) GET requests: query params and headers](#2-get-requests-query-params-and-headers)
- [3) POST requests: form vs JSON](#3-post-requests-form-vs-json)
- [4) Handling responses: status, headers, text, bytes, JSON](#4-handling-responses-status-headers-text-bytes-json)
- [5) Timeouts, errors, and retries](#5-timeouts-errors-and-retries)
- [6) Sessions: connection reuse and default headers](#6-sessions-connection-reuse-and-default-headers)
- [7) Auth: Bearer tokens, Basic, and .env](#7-auth-bearer-tokens-basic-and-env)
- [8) File upload and download](#8-file-upload-and-download)
- [9) Pagination patterns](#9-pagination-patterns)
- [10) Proxies, SSL verification, and certificates](#10-proxies-ssl-verification-and-certificates)
- [11) Encoding, compression, and content types](#11-encoding-compression-and-content-types)
- [12) Optional: async with httpx](#12-optional-async-with-httpx)
- [13) Troubleshooting](#13-troubleshooting)
- [14) Recap](#14-recap)

---

## 0) Install and quick test
**What**: Add libraries and confirm networking works.  
**Why**: Ensure your environment can reach the internet.
```sh
pip install requests httpx python-dotenv
python - <<'PY'
import requests
r = requests.get("https://httpbin.org/get", timeout=10)
print(r.status_code, r.headers.get('content-type'))
PY
```

---

## 1) HTTP basics: methods and URLs
**What**: HTTP methods describe intent.  
**Why**: Servers react differently to each method.
- **GET**: retrieve data, no body.
- **POST**: create or submit data.
- **PUT**: replace a resource.
- **PATCH**: partial update.
- **DELETE**: remove a resource.
- **HEAD/OPTIONS**: metadata or capability checks.

URLs can include **paths**, **query strings** (`?key=value`), and **fragments** (`#ignored-by-server`).

---

## 2) GET requests: query params and headers
**What**: Send read‑only requests with filters and metadata.  
**Why**: Safe, cacheable access.
```python
import requests

base = "https://httpbin.org/get"
params = {"q": "python", "page": 2}
headers = {"User-Agent": "python-notes/1.0"}

r = requests.get(base, params=params, headers=headers, timeout=10)
print(r.url)                 # full URL with encoded query
print(r.status_code)         # 200
print(r.json()["args"])     # parsed query params on the server side
```
`params=` handles URL encoding for you. Use `headers=` for auth, content negotiation, or custom identifiers.

---

## 3) POST requests: form vs JSON
**What**: Send data to the server in the request body.  
**Why**: Create resources or submit forms.
```python
import requests

# Form-encoded (application/x-www-form-urlencoded)
form_data = {"username": "ned", "password": "secret"}
r1 = requests.post("https://httpbin.org/post", data=form_data, timeout=10)
print(r1.json()["form"])  # {'username': 'ned', 'password': 'secret'}

# JSON (application/json)
payload = {"name": "Widget", "price": 9.99}
r2 = requests.post("https://httpbin.org/post", json=payload, timeout=10)
print(r2.json()["json"])  # {'name': 'Widget', 'price': 9.99}
```
Use `json=` for JSON bodies. Avoid manually `dumps` unless you need customization. For arbitrary content, use `data=` with your own `headers={'Content-Type': ...}`.

---

## 4) Handling responses: status, headers, text, bytes, JSON
**What**: Inspect results and parse content.  
**Why**: Correctly handle success vs failure.
```python
r = requests.get("https://httpbin.org/anything", timeout=10)
print(r.status_code)         # e.g., 200
print(r.headers.get("Server"))
print(r.text[:80])           # decoded text (uses r.encoding)
print(len(r.content))        # raw bytes length

# JSON helper with validation
try:
    data = r.json()
except ValueError:
    data = None
```
`r.raise_for_status()` throws for 4xx/5xx. Prefer explicit checks when APIs use non‑200 for flow control.

---

## 5) Timeouts, errors, and retries
**What**: Avoid hanging and recover from transient failures.  
**Why**: Network is unreliable.
```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util import Retry

session = requests.Session()
retries = Retry(
    total=5,
    backoff_factor=0.5,            # 0.5s, 1s, 2s, 4s, ...
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["GET", "POST", "PUT", "DELETE", "PATCH"]
)
session.mount("https://", HTTPAdapter(max_retries=retries))

r = session.get("https://httpbin.org/status/503", timeout=5)
print(r.status_code)
```
Always pass a **timeout**. Use exponential backoff for idempotent or retry‑safe requests.

---

## 6) Sessions: connection reuse and default headers
**What**: Keep TCP connections open and set shared defaults.  
**Why**: Performance and consistency.
```python
import requests

with requests.Session() as s:
    s.headers.update({"User-Agent": "python-notes/1.0"})
    # set a base URL pattern yourself if helpful
    r = s.get("https://httpbin.org/uuid", timeout=5)
    r2 = s.get("https://httpbin.org/headers", timeout=5)
    print(r.json(), r2.json()["headers"]["User-Agent"])  # same UA
```

---

## 7) Auth: Bearer tokens, Basic, and .env
**What**: Authenticate to APIs.  
**Why**: Access protected endpoints.
```python
import os
from dotenv import load_dotenv
import requests
from requests.auth import HTTPBasicAuth

load_dotenv()
API_TOKEN = os.getenv("API_TOKEN")

# Bearer / token auth
headers = {"Authorization": f"Bearer {API_TOKEN}"}
requests.get("https://httpbin.org/bearer", headers=headers, timeout=10)

# Basic auth
requests.get("https://httpbin.org/basic-auth/ned/secret",
             auth=HTTPBasicAuth("ned", "secret"), timeout=10)
```
Store secrets in `.env` and never commit them. Many APIs also accept tokens in `?access_token=` or cookies; prefer headers.

---

## 8) File upload and download
**What**: Send or receive files efficiently.  
**Why**: Common API task.
```python
import requests

# upload (multipart/form-data)
with open("logo.png", "rb") as f:
    files = {"file": ("logo.png", f, "image/png")}
    r = requests.post("https://httpbin.org/post", files=files, timeout=30)
    print(r.status_code)

# download streaming to disk
url = "https://httpbin.org/image/png"
with requests.get(url, stream=True, timeout=30) as resp:
    resp.raise_for_status()
    with open("download.png", "wb") as out:
        for chunk in resp.iter_content(chunk_size=8192):
            if chunk:
                out.write(chunk)
```
Use `stream=True` for large files to avoid loading into memory.

---

## 9) Pagination patterns
**What**: Fetch multi‑page results.  
**Why**: APIs rarely return everything at once.
```python
def fetch_all(session, base_url):
    params = {"page": 1}
    while True:
        r = session.get(base_url, params=params, timeout=10)
        r.raise_for_status()
        data = r.json()
        yield from data["items"]
        if not data.get("next_page"):
            break
        params["page"] = data["next_page"]

import requests
with requests.Session() as s:
    for item in fetch_all(s, "https://example.com/api/items"):
        print(item)
```
Adjust for cursor‑based APIs by passing and updating `cursor` from the response.

---

## 10) Proxies, SSL verification, and certificates
**What**: Route traffic and control TLS.  
**Why**: Corporate networks or custom CAs.
```python
proxies = {
    "http": "http://proxy.corp:8080",
    "https": "http://proxy.corp:8080",
}

# verify can be a bool or a path to a CA bundle
r = requests.get("https://httpbin.org/get", proxies=proxies, verify=True, timeout=10)
```
If your org uses a custom root CA, point `verify` to the bundle or set `REQUESTS_CA_BUNDLE` env var.

---

## 11) Encoding, compression, and content types
**What**: Control how bytes become strings and how payloads are compressed.  
**Why**: Avoid mojibake and save bandwidth.
- `r.encoding`: guessed from headers; set manually if wrong.
- `Accept-Encoding: gzip, br` is automatic; `requests` decompresses by default.
- Always check `Content-Type` to choose parser: `application/json`, `text/html`, `text/csv`, etc.
```python
r = requests.get("https://httpbin.org/encoding/utf8", timeout=10)
r.encoding = "utf-8"   # override if needed
text = r.text
```

---

## 12) Optional: async with httpx
**What**: Concurrency for many requests.  
**Why**: Higher throughput for I/O‑bound tasks.
```python
import asyncio
import httpx

async def main():
    async with httpx.AsyncClient(http2=True, timeout=10.0) as client:
        urls = ["https://httpbin.org/get" for _ in range(5)]
        resps = await asyncio.gather(*(client.get(u) for u in urls))
        print([r.status_code for r in resps])

asyncio.run(main())
```
`httpx` mirrors the `requests` API and supports HTTP/2, timeouts, retries (via transport), and async.

---

## 13) Troubleshooting
- **Hangs forever**: you forgot `timeout`. Always pass it.
- **`JSONDecodeError`**: response isn’t JSON or has invalid syntax; inspect `r.text` and `Content-Type`.
- **`SSLError`**: missing corporate CA; set `REQUESTS_CA_BUNDLE` or `verify=path/to/ca.pem`.
- **`429 Too Many Requests`**: backoff and respect `Retry-After` header.
- **`ConnectionError`**: proxy required; set `proxies` or `HTTP(S)_PROXY` env vars.

---

## 14) Recap
```plaintext
Choose method → Build URL/params/headers → Send with timeout → Check status → Parse content (JSON/text/bytes) → Handle errors → Reuse session → Add auth/retries as needed
```

**Next**: Combine with `004_WEBSERVER.md` to build APIs, and use this guide to consume them. Add logging, metrics, and structured error handling for production.