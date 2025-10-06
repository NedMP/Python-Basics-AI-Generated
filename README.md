

# ⚠️ AI-Generated Documentation

> This repository’s documentation and structure were generated and maintained with AI assistance.

---

## 🧭 Project Overview
This repository is a **comprehensive Python reference and learning series** designed for:
- **IT administrators and automation engineers** working with macOS, Windows, and cloud environments.
- **Sysadmins and support specialists** who want practical, reusable Python scripts.
- **Learners returning to Python**, needing a structured, modern roadmap covering environment setup through automation, APIs, and cloud scripting.

Each numbered file is a self-contained guide covering one Python topic, written to be clear, modern, and immediately useful for real-world automation.

---

## 📂 Repository Structure

| File | Description | Status |
|------|--------------|:------:|
| 000_STRUCTURE.md | Style and format guide for all documents | ✅ |
| 001_OVERVIEW.md | Project overview and roadmap | ✅ |
| 002_SETUP.md | Environment setup, pyenv, venv, and requirements | ✅ |
| 003_BASICS.md | Python language fundamentals | ✅ |
| 004_WEBSERVER.md | Web server setup and routing (FastAPI) | ✅ |
| 005_REQUESTS.md | Making web requests and handling responses | ✅ |
| 006_SCRAPING.md | Web scraping with BeautifulSoup and Selenium | ✅ |
| 007_TKINTER.md | UI development using tkinter | ✅ |
| 008_PYGAME.md | Game loops and event handling with pygame | ✅ |
| 009_THREADING.md | Multithreading and concurrency fundamentals | ✅ |
| 010_OCR.md | Optical Character Recognition (OCR) with Tesseract | ✅ |
| 011_GRAPHS.md | Data visualization and interactive graphing | ✅ |
| 012_UIAUTOMATION.md | Keyboard/mouse/device automation (pyautogui) | ✅ |
| 013_CSVEXCEL.md | Working with CSV and Excel files | ✅ |
| 014_EMAILS.md | Sending and automating emails (Gmail + Graph) | ✅ |
| 015_FILESYSTEM.md | Filesystem I/O, safe deletes, and monitoring | ✅ |
| 016_LOGGING.md | Logging, rotating logs, and JSON structured logging | ✅ |
| 017_SUBPROCESS.md | Running system commands safely and asynchronously | ✅ |
| 018_NETWORKING.md | Networking basics, sockets, ping, and diagnostics | ✅ |
| 019_SSH_AUTOMATION.md | Remote SSH/SFTP automation with Paramiko/Fabric | ✅ |
| 020_API_AUTOMATION.md | REST API scripting, pagination, and rate-limits | ✅ |
| 021_JSON_YAML_TOML.md | Working with config file formats (JSON/YAML/TOML) | ✅ |
| 022_AZURE_AUTOMATION.md | Azure resource management with Python SDK | ✅ |
| 023_AWS_AUTOMATION.md | AWS automation using boto3 | ✅ |
| 024_SHELL_INTERACTION.md | Shell/Python interoperability and scripting | ✅ |
| 025_MONITORING_SCRIPTS.md | Health checks and alert scripts | ✅ |
| 026_TASK_SCHEDULING.md | Task scheduling (schedule, cron, APScheduler) | ✅ |
| 027_TEXT_PARSING.md | Regex, log parsing, and text analysis | ✅ |
| 028_PDF_DOC_AUTOMATION.md | Automating PDF manipulation | ✅ |
| 029_DATA_CLEANUP.md | Cleaning and validating structured data | ✅ |
| 030_BACKUP_SCRIPTS.md | Backup automation and verification | ✅ |
| 031_NOTIFICATION_AUTOMATION.md | Slack, Teams, and webhook notifications | ✅ |
| 032_BROWSER_AUTOMATION.md | Browser automation with Playwright/Selenium | ✅ |
| 033_KEYCHAIN_SECRETS.md | Secure credential handling | ✅ |
| 034_FILE_UPLOAD_DOWNLOAD.md | File transfers (HTTP, SFTP, Cloud) | ✅ |
| 035_SCRIPT_PACKAGING.md | Packaging Python tools and CLIs | ✅ |
| 036_DOCKER_AUTOMATION.md | Managing Docker via Python SDK | ✅ |
| 037_GIT_AUTOMATION.md | Git automation with GitPython | ✅ |
| 038_UNITTEST_AUTOMATION.md | Testing and CI integration | ✅ |
| 039_SYSTEM_INFO.md | Gathering system metrics (CPU, memory, disk) | ✅ |
| 040_PERSONAL_AUTOMATION.md | Personal/macOS automation utilities | ✅ |
| 041_EMAIL_REPORTING.md | Automated report generation and email | ✅ |
| 042_AI_AUTOMATION.md | Using AI/LLM APIs for automation and parsing | ✅ |
| 043_EVENT_LOG_PARSING.md | Parsing system and application event logs | ✅ |
| 044_NETWORK_INVENTORY.md | Scanning and inventorying network devices | ✅ |

✅ = Complete | ❌ = In Progress / Planned

---

## 🧠 How to Use This Repository
1. Start with **002_SETUP.md** to configure Python and environment management.  
2. Progress sequentially through **003–010** for language fundamentals.  
3. From **011+**, topics become specialized (automation, APIs, cloud).  
4. Each document is standalone but follows a consistent pattern for easy cross‑reference.

---

## 🔧 Future Plans
- Add automated testing for all code samples (pytest) and run them in CI.
- Provide a script index and runnable snippets in a `scripts/` directory.
- Cross‑link related guides and add a searchable index.

---

## 📝 Prompt Template for Adding or Updating Guides

Use this prompt to create or revise any single guide while preserving the style rules in `000_STRUCTURE.md` and using `002_SETUP.md` for reference.

```text
!!IMPORTANT!! ONLY update <TARGET_FILENAME>! 000_STRUCTURE.md is to be treated as instructions. 002_SETUP.md can be used as additional reference:

<TARGET_FILENAME> overview - <one-line purpose of the guide>
```

**How to use**
- Replace `<TARGET_FILENAME>` with the exact Markdown filename, e.g., `045_EXAMPLE_TOPIC.md`.
- Replace the overview line with a concise, single‑sentence description of the guide’s scope and goals.
- Keep the rest of the repository unchanged. The assistant should only modify the specified file.
- Follow the layout, tone, and numbering rules from `000_STRUCTURE.md`.

---

## 🤖 Attribution
AI-generated and maintained using OpenAI’s GPT‑5 for technical drafting, structured documentation, and cross-file consistency.