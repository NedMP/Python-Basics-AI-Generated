

# Python Web Scraping Fundamentals — updated Oct 5, 2025

Beginner‑friendly guide to scraping web pages and APIs. Covers ethics and legality, HTTP basics, parsing HTML with CSS selectors and XPath, pagination, sessions and logins, structured data, dynamic pages with Playwright, and reliability tips. Use with `002_SETUP.md` for environment setup.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13.  
- Run inside an activated virtual environment.  
- Prefer legal and ethical scraping. Respect site terms, `robots.txt`, and rate limits.

---

## Table of Contents
- [0) Ethics, legality, and `robots.txt`](#0-ethics-legality-and-robotstxt)
- [1) Choosing a tool](#1-choosing-a-tool)
- [2) Quick start: fetch + parse](#2-quick-start-fetch--parse)
- [3) CSS selectors and XPath](#3-css-selectors-and-xpath)
- [4) Extracting common patterns](#4-extracting-common-patterns)
- [5) Pagination](#5-pagination)
- [6) Sessions, cookies, and logins](#6-sessions-cookies-and-logins)
- [7) Structured data: JSON in pages and APIs](#7-structured-data-json-in-pages-and-apis)
- [8) Dynamic pages: Playwright](#8-dynamic-pages-playwright)
- [9) Files: images, PDFs, and downloads](#9-files-images-pdfs-and-downloads)
- [10) Performance and reliability](#10-performance-and-reliability)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Ethics, legality, and `robots.txt`
**What**: Determine if you may scrape a site and how to do so responsibly.  
**Why**: To avoid legal issues and service disruption.
- Read the site’s **Terms of Service**. Some prohibit scraping.  
- Check `https://example.com/robots.txt` for crawl rules. Robots is advisory for bots and often lists disallowed paths.  
- Rate limit requests. Start with 1 req/s or slower.  
- Identify yourself with a descriptive `User-Agent` and contact if appropriate.  
- Never attempt to bypass authentication, paywalls, or technical measures.  
- Prefer **official APIs** when available.

---

## 1) Choosing a tool
- **requests + BeautifulSoup (bs4)**: simplest sync HTML parsing.
- **httpx**: modern sync/async HTTP (HTTP/2).  
- **lxml / selectolax / parsel**: faster parsers or XPath support.  
- **Playwright**: headless browser for JS‑rendered pages and interactions.  
- **Selenium**: similar to Playwright; Playwright tends to be simpler and faster in 2025.

Install a solid default set:
```sh
pip install requests httpx beautifulsoup4 lxml selectolax parsel python-dotenv
pip install playwright  # for dynamic sites
playwright install      # one-time browser download
```

---

## 2) Quick start: fetch + parse
**Goal**: Download an HTML page and extract items.
```python
import requests
from bs4 import BeautifulSoup

url = "https://httpbin.org/html"
headers = {"User-Agent": "python-notes-scraper/1.0"}

r = requests.get(url, headers=headers, timeout=15)
r.raise_for_status()

soup = BeautifulSoup(r.text, "lxml")  # or "html.parser"
print(soup.title.text.strip())
```
Notes:
- Always set a **timeout** and call `raise_for_status()`.
- Prefer `lxml` parser for speed and correctness.

---

## 3) CSS selectors and XPath
**CSS selectors** with BeautifulSoup:
```python
soup.select("div.card a.title")          # all <a class="title"> in a .card
soup.select_one("ul#products > li:nth-of-type(1)")
[item.get_text(strip=True) for item in soup.select(".price")]  # text list
```
**XPath** with `lxml` or `parsel`:
```python
from parsel import Selector
sel = Selector(r.text)
sel.xpath('//div[@class="card"]//a[@class="title"]/text()').getall()
sel.xpath('string(//h1)').get()  # normalized full text
```
Which to use? CSS is usually simpler. XPath is powerful for complex conditions and DOM traversal.

---

## 4) Extracting common patterns
**Links and attributes**
```python
for a in soup.select('a[href]'):
    href = a['href']
    text = a.get_text(strip=True)
```
**Tables**
```python
rows = []
for tr in soup.select('table tr'):
    cells = [td.get_text(strip=True) for td in tr.select('th, td')]
    if cells: rows.append(cells)
```
**Dates and numbers**
```python
import re
text = soup.get_text(" ", strip=True)
prices = re.findall(r"\$\s?([0-9]+(?:\.[0-9]{2})?)", text)
```
**Normalization**
```python
from urllib.parse import urljoin
base = "https://example.com/"
abs_links = [urljoin(base, a['href']) for a in soup.select('a[href]')]
```

---

## 5) Pagination
**Offset/page model**
```python
import requests, time

session = requests.Session()
session.headers.update({"User-Agent": "python-notes-scraper/1.0"})

page = 1
while True:
    r = session.get("https://example.com/list", params={"page": page}, timeout=15)
    r.raise_for_status()
    soup = BeautifulSoup(r.text, "lxml")
    items = soup.select(".item")
    if not items:
        break
    # process items ...
    page += 1
    time.sleep(1.0)  # be nice
```
**Link‑driven**: follow `Next` link with `rel="next"` or `.pagination a:contains("Next")` depending on site.

---

## 6) Sessions, cookies, and logins
```python
import requests
from bs4 import BeautifulSoup

s = requests.Session()
s.headers.update({"User-Agent": "python-notes-scraper/1.0"})

# 1) Get login page to fetch cookies or CSRF token
r = s.get("https://example.com/login", timeout=15)
soup = BeautifulSoup(r.text, "lxml")
csrf = soup.select_one('input[name="csrf_token"]').get('value')

# 2) Post credentials
payload = {"username": "ned", "password": "secret", "csrf_token": csrf}
s.post("https://example.com/session", data=payload, timeout=15)

# 3) Authenticated request reuses cookies automatically
resp = s.get("https://example.com/account", timeout=15)
```
If the site requires JavaScript for login flows, use Playwright and `page.fill()/page.click()`.

---

## 7) Structured data: JSON in pages and APIs
**HTML‑embedded JSON** (e.g., `<script type="application/ld+json">`):
```python
import json
for node in soup.select('script[type="application/ld+json"]'):
    data = json.loads(node.string)
```
**Hidden JSON endpoints**: inspect Network tab in DevTools. Many sites call JSON APIs your code can use directly.
```python
import requests
r = requests.get("https://example.com/api/products", params={"q": "laptop"}, timeout=15)
data = r.json()["items"]
```

---

## 8) Dynamic pages: Playwright
**What**: Render JavaScript, click buttons, scroll, and intercept network calls.
```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page(user_agent="python-notes-scraper/1.0")
    page.goto("https://example.com", wait_until="networkidle")
    # click a button and wait for results to load
    page.click("text=Load more")
    page.wait_for_selector(".result-item")
    html = page.content()
    # parse with BeautifulSoup
```
**Waits**: prefer `wait_for_selector()` and `wait_until="networkidle"` over `sleep()`.

---

## 9) Files: images, PDFs, and downloads
```python
import requests
from pathlib import Path

url = "https://example.com/file.pdf"
with requests.get(url, stream=True, timeout=60) as r:
    r.raise_for_status()
    Path("downloads").mkdir(exist_ok=True)
    with open("downloads/file.pdf", "wb") as f:
        for chunk in r.iter_content(8192):
            if chunk:
                f.write(chunk)
```
Set a reasonable chunk size and always check `Content-Type` and file sizes.

---

## 10) Performance and reliability
- **Sessions**: reuse connections with `requests.Session()` or `httpx.Client()`.
- **Backoff**: use exponential backoff on 429/5xx. Respect `Retry-After`.
- **Caching**: persist responses you’ve already fetched when iterating locally.
- **Parsing speed**: `selectolax` is faster than BeautifulSoup for heavy scraping.
- **Throttling**: insert `time.sleep()` or token‑bucket logic to avoid bans.
- **Randomization**: vary delays and order if appropriate, without evading legal controls.
- **Proxies**: only if permitted by policy; configure `HTTP(S)_PROXY` env vars.

Minimal retry helper with `requests`:
```python
from requests.adapters import HTTPAdapter
from urllib3.util import Retry
s = requests.Session()
s.mount("https://", HTTPAdapter(max_retries=Retry(total=5, backoff_factor=0.5,
    status_forcelist=[429,500,502,503,504], allowed_methods=["GET","POST"])) )
```

---

## 11) Troubleshooting
- **Empty content**: site is JS‑rendered → use Playwright or find the JSON API.
- **403/429**: missing headers or too fast; set `User-Agent`, add delays, handle retries.
- **Encoding issues**: set `r.encoding` before `r.text` or parse bytes.
- **Broken selectors**: DOM changed; inspect again and prefer stable attributes (`data-*`).
- **Blocked by robots**: respect it; don’t bypass. Seek permission or official API.
- **Login fails**: CSRF token missing or wrong form field names.

---

## 12) Recap
```plaintext
Decide legality → Pick tool → Fetch with timeout → Parse via CSS/XPath → Handle pagination → Maintain session/login → Prefer JSON when possible → Use Playwright for JS → Add retries and rate limits
```

**Next**: Add persistence (SQLite/CSV), structured logging, and schedule jobs responsibly (cron or Airflow).