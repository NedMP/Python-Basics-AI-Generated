

# Python Notes Overview & Roadmap — updated Oct 5, 2025

High‑level index for the `Python_Notes` collection. Use this to navigate the existing guides, see what each file covers at a glance, and plan what to read next. Follows the house style (see `000_STRUCTURE.md`) while serving as a directory rather than a tutorial.

---

**Assumptions**  
- macOS, zsh, Python 3.12–3.13 in a virtual environment.  
- Practical, copy‑pasteable material with short explanations first.

---

## Table of Contents
- [0) How to use this overview](#0-how-to-use-this-overview)
- [1) Core learning path (recommended order)](#1-core-learning-path-recommended-order)
- [2) Existing files (000 and 002–014)](#2-existing-files-000-and-002–014)
- [3) Planned additions (015–044)](#3-planned-additions-015–044)
- [4) Conventions and cross‑refs](#4-conventions-and-cross-refs)
- [5) Maintenance & versioning](#5-maintenance--versioning)
- [6) Recap flow](#6-recap-flow)

---

## 0) How to use this overview
**What**: A map of topics with brief, task‑oriented summaries.  
**Why**: Reduces search time and makes the series cohesive.  
**How**: Skim Section 1 to follow the recommended path, then jump to detailed guides via links in Sections 2–3.

---

## 1) Core learning path (recommended order)
Read in this sequence for a smooth ramp from environment → programming → automation:

1. **002_SETUP.md** — Install Python with Homebrew, create venvs, `.env` files, requirements, pyenv.  
2. **003_BASICS.md** — Variables, types, flow control, functions, modules, CLI basics.  
3. **005_REQUESTS.md** — HTTP fundamentals, headers, JSON, sessions, retries.  
4. **011_GRAPHS.md** — Visualize data to verify results quickly.  
5. **013_CSVEXCEL.md** — Import/export CSV/Excel for real‑world data flows.  
6. **006_SCRAPING.md** — Pull data from the web ethically; CSS/XPath; Playwright for JS.  
7. **009_THREADING.md** — Concurrency basics for I/O‑bound tasks.  
8. **004_WEBSERVER.md** — Build small APIs with FastAPI to automate workflows.  
9. **012_UIAUTOMATION.md** — Desktop automation when no API exists.  
10. **014_EMAILS.md** — Notify and integrate via Gmail/Graph.  
11. **010_OCR.md** — Extract text from scans when data isn’t digital.  
12. **007_TKINTER.md** / **008_PYGAME.md** — Optional UI/interactive modules.

---

## 2) Existing files (000 and 002–014)
Short descriptions with links. These are complete.

- **000_STRUCTURE.md** — Style guide for all docs: layout, tone, TOC rules, recap flow, validation checklist.  
- **002_SETUP.md** — Python + venv on macOS. Install Python, create venvs, manage `.env`, freeze requirements, pyenv.  
- **003_BASICS.md** — Python fundamentals and a tiny CLI app. Variables, lists/dicts, loops, functions, imports, packaging basics.  
- **004_WEBSERVER.md** — FastAPI + Uvicorn. Routes, params, models, status codes, errors, middleware, .env, tests, prod, Docker.  
- **005_REQUESTS.md** — HTTP with `requests` and optional `httpx`. Methods, params, headers, JSON, files, sessions, retries, proxies, async.  
- **006_SCRAPING.md** — Scraping ethics, requests+BS4, CSS/XPath, pagination, sessions/logins, JSON endpoints, Playwright, retries.  
- **007_TKINTER.md** — Tkinter/ttk GUI. Layout managers, widgets, events, dialogs, canvas, images, threading, packaging.  
- **008_PYGAME.md** — Pygame loop, timing with delta, input, drawing, sprites, collisions, scenes, simple physics, packaging.  
- **009_THREADING.md** — Threads vs processes vs asyncio, GIL, locks, queues, ThreadPoolExecutor, cancellation, patterns.  
- **010_OCR.md** — Tesseract/pytesseract, preprocessing with OpenCV, PDFs with OCRmyPDF, alternatives, cloud OCR, accuracy tips.  
- **011_GRAPHS.md** — Matplotlib, pandas plotting, Plotly interactivity, dates/categories, subplots, exports, mini dashboard.  
- **012_UIAUTOMATION.md** — PyAutoGUI on macOS. Permissions, safety, coordinates, mouse/keyboard, image matching, waits.  
- **013_CSVEXCEL.md** — CSV stdlib + pandas, Excel engines, formatting with xlsxwriter, encodings/dtypes, big files.  
- **014_EMAILS.md** — Gmail API/SMTP and Microsoft Graph. OAuth/device code/client credentials, HTML, attachments, retries.

---

## 3) Planned additions (015–044)
Ordered roadmap for future docs. Names and one‑liners are stable; content will follow the style guide when authored.

**Core Automation & System**  
- **015_FILESYSTEM.md** — Paths, copy/move/delete, globbing, safe temp files, file watching.  
- **016_LOGGING.md** — `logging` basics, rotation, JSON logs, structured context.  
- **017_SUBPROCESS.md** — Run shell commands, capture output, timeouts, async subprocess.  
- **018_NETWORKING.md** — Sockets, ping/port checks, simple TCP/UDP tools.  
- **019_SSH_AUTOMATION.md** — Paramiko/Fabric for remote commands and file transfers.  
- **020_API_AUTOMATION.md** — API CRUD patterns, auth, pagination, rate limits.

**System Administration / Cloud**  
- **021_JSON_YAML_TOML.md** — Read/write config formats for automation.  
- **022_AZURE_AUTOMATION.md** — Manage Azure resources with Python SDKs and identities.  
- **023_AWS_AUTOMATION.md** — Boto3 scripting for EC2/S3/Lambda.  
- **024_SHELL_INTERACTION.md** — Blend Python with shell; env, pipes, heredocs.  
- **025_MONITORING_SCRIPTS.md** — Health checks, log watchers, alerts to chat/email.  
- **026_TASK_SCHEDULING.md** — `schedule`, APScheduler, cron/launchd patterns.

**Data & File Automation**  
- **027_TEXT_PARSING.md** — Regex and log parsing; tokenization, common patterns.  
- **028_PDF_DOC_AUTOMATION.md** — Extract, split/merge, annotate, sign.  
- **029_DATA_CLEANUP.md** — Bulk rename, dedupe, column sanitation, validation.  
- **030_BACKUP_SCRIPTS.md** — Zip/tar, incremental copies, checksums, rotation.

**Integration & User Interaction**  
- **031_NOTIFICATION_AUTOMATION.md** — Slack/Teams/webhooks, desktop notifications.  
- **032_BROWSER_AUTOMATION.md** — Playwright/Selenium for portals with no APIs.  
- **033_KEYCHAIN_SECRETS.md** — macOS Keychain, `keyring`, secure env handling.  
- **034_FILE_UPLOAD_DOWNLOAD.md** — SFTP/HTTP/S3/OneDrive transfers; resumable patterns.

**DevOps & Tooling**  
- **035_SCRIPT_PACKAGING.md** — CLI apps with `argparse`/`click`, packaging and distribution.  
- **036_DOCKER_AUTOMATION.md** — Control Docker with `docker-py`, build/run/inspect.  
- **037_GIT_AUTOMATION.md** — GitPython for repo tasks, hooks, CI helpers.  
- **038_UNITTEST_AUTOMATION.md** — Test structure for scripts, fixtures, CI usage.  
- **039_SYSTEM_INFO.md** — System inventory: CPU/RAM/disk/procs with `psutil`.

**Personal & Cross‑Use**  
- **040_PERSONAL_AUTOMATION.md** — Local workflow hacks: downloads, renames, reminders.  
- **041_EMAIL_REPORTING.md** — Generate and email daily/weekly reports from data.  
- **042_AI_AUTOMATION.md** — Use LLMs for summaries, log triage, code assist.  
- **043_EVENT_LOG_PARSING.md** — Syslog, Windows Event Log, audit extracts.  
- **044_NETWORK_INVENTORY.md** — Scan and inventory devices; export to CSV.

---

## 4) Conventions and cross‑refs
- Each file stands alone and links out only when it improves clarity.  
- Code blocks are copy‑pasteable. Short explanations precede commands.  
- `.env` files hold secrets; never commit them.  
- Cross‑reference nearby topics at the end of each doc (e.g., `005_REQUESTS.md` ↔ `020_API_AUTOMATION.md`).

---

## 5) Maintenance & versioning
- Append an **updated** date in the title when you materially change a doc.  
- Keep versions consistent with current Python LTS (3.12/3.13).  
- Pin critical package versions in examples where behavior changes across major releases.  
- Prefer stable libraries and official SDKs for enterprise topics.

---

## 6) Recap flow
```plaintext
Pick a task → Find the topic number below → Jump into the focused guide → Ship a small working script → Iterate with nearby topics when needed
```

**Next**: Start at **002_SETUP.md**, then follow the **Core learning path** above. When you hit a real‑world task, jump to the matching topic from Sections 2–3.