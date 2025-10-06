

# Browser Automation with Python (Playwright & Selenium) — updated Oct 6, 2025

Automate browsers to log into portals, submit forms, and script repetitive workflows. This guide prefers **Playwright** for reliability and speed and includes **Selenium** equivalents where useful. Covers setup, selectors and waits, login/session reuse, file upload/download, screenshots/PDF, headless CI runs, and robust patterns. Pair with `002_SETUP.md` for venv and `.env` management.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- You can create test accounts or use sandboxes.  
- Secrets live in `.env` and are not committed.

**Install**
```sh
pip install playwright python-dotenv
# install browser binaries once
python -m playwright install
# optional: only specific browsers
python -m playwright install chromium

# Selenium alternative
pip install selenium webdriver-manager python-dotenv
```

---

## Table of Contents
- [0) Choosing Playwright vs Selenium](#0-choosing-playwright-vs-selenium)
- [1) Quick start: Playwright login + form submit](#1-quick-start-playwright-login--form-submit)
- [2) Selectors, waits, and reliability](#2-selectors-waits-and-reliability)
- [3) Reusing sessions: storage state and persistent context](#3-reusing-sessions-storage-state-and-persistent-context)
- [4) Forms: inputs, dropdowns, file upload](#4-forms-inputs-dropdowns-file-upload)
- [5) Downloads, screenshots, and PDF export](#5-downloads-screenshots-and-pdf-export)
- [6) Scraping tables and APIs behind auth](#6-scraping-tables-and-apis-behind-auth)
- [7) 2FA/MFA flows and TOTP](#7-2famfa-flows-and-totp)
- [8) Headless, CI, and containers](#8-headless-ci-and-containers)
- [9) Selenium equivalents](#9-selenium-equivalents)
- [10) Robust scripting patterns](#10-robust-scripting-patterns)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Choosing Playwright vs Selenium
**What**: Pick the right automation stack.  
**Why**: Playwright ships with auto‑waits, fast selectors, and bundled browsers; Selenium is ubiquitous and integrates anywhere.
- **Playwright**: auto‑waiting, modern selectors (`get_by_role`, `:has`, `Locator`), trace viewer, storage state. Preferred for reliability.
- **Selenium**: widest language support and legacy compatibility; good when Playwright is not allowed.

---

## 1) Quick start: Playwright login + form submit
**What**: Minimal script to load creds from `.env`, log in, and submit a form.
```python
# file: pw_login.py
from dotenv import load_dotenv; load_dotenv()
import os
from pathlib import Path
from playwright.sync_api import sync_playwright

BASE = os.getenv("BASE_URL", "https://example.com")
USER = os.getenv("USER_EMAIL", "user@example.com")
PASS = os.getenv("USER_PASSWORD", "secret")

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)  # set True in CI
    ctx = browser.new_context()
    page = ctx.new_page()

    page.goto(f"{BASE}/login", wait_until="networkidle")
    page.get_by_label("Email").fill(USER)
    page.get_by_label("Password").fill(PASS)
    page.get_by_role("button", name="Sign in").click()

    # wait for post-login element
    page.get_by_role("heading", name="Dashboard").wait_for()

    # simple form
    page.get_by_placeholder("Search").fill("Quarterly report")
    page.get_by_role("button", name="Search").click()

    page.screenshot(path="after_login.png", full_page=True)
    ctx.close(); browser.close()
```
Run:
```sh
BASE_URL=https://your-portal.example.com USER_EMAIL=me@example.com USER_PASSWORD='...' python pw_login.py
```

---

## 2) Selectors, waits, and reliability
**What**: Use resilient selectors and built‑in waits to avoid flakiness.
```python
# Prefer role/label/placeholder-based selectors
page.get_by_role("button", name="Save").click()
page.get_by_label("Workspace name").fill("Ops")
page.get_by_placeholder("Search").press("Enter")

# Locator API: chain and scope
row = page.get_by_role("row", name="Invoice 12345")
row.get_by_role("button", name="Approve").click()

# Explicit waiting when needed
page.wait_for_url("**/dashboard")
page.wait_for_load_state("networkidle")
```
Tips:
- Avoid brittle CSS like `div:nth-child(3) > span`.  
- Prefer `get_by_role` for accessibility‑aware apps.  
- Use `locator = page.locator("css=...")` when ARIA is unavailable.

---

## 3) Reusing sessions: storage state and persistent context
**What**: Keep users logged in across runs to skip login steps.

### 3.1 Export storage state after login
```python
ctx = browser.new_context()
page = ctx.new_page()
# ...perform login...
ctx.storage_state(path="state.json")
```

### 3.2 Import state on next run
```python
ctx = browser.new_context(storage_state="state.json")
page = ctx.new_page()
page.goto(BASE + "/dashboard")
```

### 3.3 Persistent context (profile dir)
```python
user_data_dir = "/tmp/pw-profile"
browser = p.chromium.launch_persistent_context(user_data_dir, headless=True)
page = browser.new_page()
```
Notes:
- Storage state contains cookies and localStorage; treat as sensitive.  
- Rotate state when SSO policies expire.

---

## 4) Forms: inputs, dropdowns, file upload
**What**: Fill complex forms reliably.
```python
# text inputs and selects
page.get_by_label("First name").fill("Ned")
page.get_by_label("Country").select_option("AU")

# checkboxes and radios
page.get_by_label("Accept terms").check()
page.get_by_label("Plan Pro").check()

# file upload
from pathlib import Path
page.set_input_files("input[type=file]", Path("/path/to/report.csv"))

# date pickers often need direct input
page.fill("input[name=date]", "2025-10-06")
```

---

## 5) Downloads, screenshots, and PDF export
**What**: Capture artifacts for audits and hand‑offs.
```python
# downloads
with page.expect_download() as d:
    page.get_by_role("button", name="Export").click()
download = d.value
download.save_as("export.csv")

# screenshots
page.screenshot(path="screen.png", full_page=True)

# PDF (Chromium-only)
page.pdf(path="page.pdf", format="A4")
```

---

## 6) Scraping tables and APIs behind auth
**What**: Extract rows after login; or call in‑page APIs with the session context.
```python
# grab visible table text
rows = page.locator("table tbody tr")
items = []
for i in range(rows.count()):
    tds = rows.nth(i).locator("td")
    items.append({
        "id": tds.nth(0).inner_text().strip(),
        "name": tds.nth(1).inner_text().strip(),
        "status": tds.nth(2).inner_text().strip(),
    })

# fetch with page context (cookies included)
resp = page.request.get(BASE + "/api/items")
print(resp.status, resp.json()[:2])
```
Write results to CSV/Excel per `013_CSVEXCEL.md`.

---

## 7) 2FA/MFA flows and TOTP
**What**: Handle MFA during login.
- **Best**: Manually seed `state.json` once by logging in interactively, then reuse.  
- **TOTP**: If policy allows, generate codes with `pyotp`.
```sh
pip install pyotp
```
```python
import pyotp, os
secret = os.getenv("TOTP_SECRET")  # store in .env or Keychain
code = pyotp.TOTP(secret).now()
page.get_by_label("Authentication code").fill(code)
```
Caution: Do not bypass security; follow org policies.

---

## 8) Headless, CI, and containers
**What**: Run without UI for schedulers and CI.
```python
browser = p.chromium.launch(headless=True, args=["--no-sandbox"])  # in CI
```
Artifacts:
- Save screenshots/PDF on failure.  
- Capture Playwright traces:
```python
ctx = browser.new_context()
ctx.tracing.start(screenshots=True, snapshots=True)
# ... run steps ...
ctx.tracing.stop(path="trace.zip")
```

---

## 9) Selenium equivalents
**What**: Minimal examples if you must use Selenium.
```python
# file: sel_login.py
from dotenv import load_dotenv; load_dotenv()
import os
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

BASE = os.getenv("BASE_URL"); USER = os.getenv("USER_EMAIL"); PASS = os.getenv("USER_PASSWORD")

opts = webdriver.ChromeOptions()
# opts.add_argument("--headless=new")
driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=opts)

driver.get(f"{BASE}/login")
WebDriverWait(driver, 20).until(EC.presence_of_element_located((By.NAME, "email"))).send_keys(USER)
driver.find_element(By.NAME, "password").send_keys(PASS)
driver.find_element(By.XPATH, "//button[contains(., 'Sign in')]").click()
WebDriverWait(driver, 20).until(EC.url_contains("/dashboard"))

driver.save_screenshot("sel_after.png")
driver.quit()
```

---

## 10) Robust scripting patterns
**What**: Keep scripts maintainable and safe.
- Centralize URLs, timeouts, and selectors.  
- Use **env vars** for creds; never hardcode.  
- Add retries for flaky endpoints; log timings.  
- Prefer role/label selectors; fall back to stable `data-testid` attributes.  
- Wrap actions with small helpers (e.g., `click_and_wait`).  
- Use **context managers** for downloads and traces.

Structure:
```
auto/
├── pw_login.py
├── lib/
│   ├── auth.py
│   └── pages.py
└── .env
```

---

## 11) Troubleshooting
- **`TimeoutError`**: selector never appears. Re-check role/label, increase timeout, or wait for network idle.  
- **Blocked by robots or WAF**: respect site rules; consider official APIs.  
- **MFA loops**: seed storage state interactively; avoid automating protected steps if policy forbids.  
- **Downloads hang**: wrap with `expect_download`.  
- **Headless rendering differences**: try non‑headless or set viewport and user agent.  
- **Selenium driver mismatch**: use `webdriver-manager` or keep Chrome/driver versions aligned.

---

## 12) Recap
```plaintext
Pick Playwright → install browsers → write resilient selectors with auto-waits → login and save storage state → automate forms/uploads → capture downloads/screens → run headless in CI → use Selenium only where required
```

**Next**: Combine with `025_MONITORING_SCRIPTS.md` for watched jobs, `026_TASK_SCHEDULING.md` to run nightly, and `029_DATA_CLEANUP.md` to process downloaded reports. 