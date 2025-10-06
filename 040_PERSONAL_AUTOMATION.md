

# Personal Automation with Python — updated Oct 6, 2025

Automate everyday tasks on your Mac: **file organization**, **reminders/notifications**, **clipboard utilities**, and small **productivity scripts**. This note gives ready‑to‑run snippets plus a simple structure so you can extend them. Pair with `002_SETUP.md` for venv, `.env`, and general setup.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Scripts live under `~/Code_Stuff/automation/` and are run manually, scheduled, or bound to hotkeys.  
- Secrets and config via `.env` or per‑script `config.toml`.

**Install**
```sh
pip install python-dotenv watchdog click pyperclip rich tomli
```
- `watchdog` for filesystem watching.  
- `click` for ergonomic CLIs.  
- `pyperclip` for clipboard access (falls back to `pbcopy/pbpaste`).  
- `tomli` to read TOML config on Python <3.11 (stdlib `tomllib` on 3.11+).

---

## Table of Contents
- [0) Structure and safety](#0-structure-and-safety)
- [1) Download folder organizer](#1-download-folder-organizer)
- [2) Real-time watcher (watchdog)](#2-real-time-watcher-watchdog)
- [3) Quick reminders and notifications](#3-quick-reminders-and-notifications)
- [4) Clipboard utilities](#4-clipboard-utilities)
- [5) Text filters: slugify, URL-encode, case tools](#5-text-filters-slugify-url-encode-case-tools)
- [6) Tiny launchers and hotkeys](#6-tiny-launchers-and-hotkeys)
- [7) Scheduling jobs (launchd, cron, Python)](#7-scheduling-jobs-launchd-cron-python)
- [8) Putting it together: daily tidy pipeline](#8-putting-it-together-daily-tidy-pipeline)
- [9) Troubleshooting](#9-troubleshooting)
- [10) Recap](#10-recap)

---

## 0) Structure and safety
**What**: A predictable layout and non‑destructive defaults.  
**Why**: Prevents accidental data loss and simplifies reuse.
```
automation/
├── .env
├── organizer.py
├── watcher.py
├── remind.py
├── clipboard.py
├── textfilters.py
└── jobs/
    └── daily_tidy.py
```
Principles:
1. **Copy/Move to staged folders**, never delete outright.  
2. **Logs** with counts and actions.  
3. **Dry‑run** flag before first real pass.

---

## 1) Download folder organizer
**What**: Sort files in `~/Downloads` into subfolders by type and age.  
**Why**: Keeps Downloads from becoming a junk drawer.
```python
# file: organizer.py
from __future__ import annotations
from pathlib import Path
import shutil, time, argparse

MAP = {
  "Images": {".png",".jpg",".jpeg",".gif",".heic"},
  "PDF": {".pdf"},
  "Docs": {".txt",".md",".docx",".pages"},
  "Archives": {".zip",".tar",".gz",".rar"},
  "Audio": {".mp3",".m4a",".wav"},
  "Video": {".mp4",".mov",".mkv"},
  "Code": {".py",".sh",".js",".json",".yaml",".yml"},
}

EXCLUDE = {".DS_Store"}

def route(p: Path) -> str:
    ext = p.suffix.lower()
    for folder, exts in MAP.items():
        if ext in exts: return folder
    return "Other"

def run(src: Path, dest: Path, days: int, dry: bool=False) -> int:
    moved = 0
    cutoff = time.time() - days*86400 if days else None
    dest.mkdir(parents=True, exist_ok=True)
    for f in src.iterdir():
        if not f.is_file() or f.name in EXCLUDE: continue
        if cutoff and f.stat().st_mtime > cutoff:  # keep very recent files
            continue
        folder = route(f)
        target_dir = dest / folder
        target_dir.mkdir(parents=True, exist_ok=True)
        target = target_dir / f.name
        i = 1
        while target.exists():
            target = target_dir / f"{f.stem}_{i}{f.suffix}"; i += 1
        print(("DRY " if dry else "") + f"move {f} -> {target}")
        if not dry:
            shutil.move(str(f), str(target))
            moved += 1
    return moved

if __name__ == "__main__":
    ap = argparse.ArgumentParser(description="Organize Downloads")
    ap.add_argument("--src", default=str(Path.home()/"Downloads"))
    ap.add_argument("--dest", default=str(Path.home()/"Downloads_Organized"))
    ap.add_argument("--older-than-days", type=int, default=1)
    ap.add_argument("--dry", action="store_true")
    a = ap.parse_args()
    count = run(Path(a.src), Path(a.dest), a.older_than_days, a.dry)
    print(f"moved: {count}")
```
Run:
```sh
python organizer.py --dry
python organizer.py --older-than-days 2
```

---

## 2) Real-time watcher (watchdog)
**What**: React instantly to new files in Downloads.  
**Why**: Auto‑sort screenshots, invoices, or reports as they arrive.
```python
# file: watcher.py
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
from pathlib import Path
import time, organizer

class Handler(FileSystemEventHandler):
    def on_created(self, event):
        if event.is_directory: return
        # wait for file to finish writing
        time.sleep(0.5)
        organizer.run(Path(event.src_path).parent, Path.home()/"Downloads_Organized", days=0, dry=False)

if __name__ == "__main__":
    path = Path.home()/"Downloads"
    obs = Observer(); obs.schedule(Handler(), str(path), recursive=False)
    obs.start(); print("watching", path)
    try:
        while True: time.sleep(5)
    except KeyboardInterrupt:
        obs.stop(); obs.join()
```

---

## 3) Quick reminders and notifications
**What**: Fire a local notification or timed reminder without extra apps.  
**Why**: Lightweight nudges for tasks and context switches.

### 3.1 Instant macOS notification (via AppleScript)
```python
# file: remind.py
import subprocess, argparse

def notify(title: str, text: str):
    script = f'display notification "{text}" with title "{title}"'
    subprocess.run(["osascript","-e", script], check=True)

if __name__ == "__main__":
    ap = argparse.ArgumentParser(); ap.add_argument("message"); ap.add_argument("--title", default="Reminder")
    a = ap.parse_args(); notify(a.title, a.message)
```
Run:
```sh
python remind.py "Stand up and stretch"
```

### 3.2 Timed reminder (sleep + notify)
```sh
# 25-minute pomodoro
sleep $((25*60)); python remind.py "Break time"
```
For recurring schedules, see Section 7.

---

## 4) Clipboard utilities
**What**: Read/transform/write clipboard contents.  
**Why**: Speed up repetitive edits.
```python
# file: clipboard.py
from __future__ import annotations
import pyperclip, argparse, urllib.parse

def read() -> str:
    return pyperclip.paste() or ""

def write(s: str) -> None:
    pyperclip.copy(s)

if __name__ == "__main__":
    ap = argparse.ArgumentParser(description="Clipboard tools")
    ap.add_argument("action", choices=["show","urlencode","urldecode","strip","lower","upper"])
    a = ap.parse_args()
    text = read()
    if a.action == "show": print(text)
    elif a.action == "urlencode": write(urllib.parse.quote(text)); print("encoded → clipboard")
    elif a.action == "urldecode": write(urllib.parse.unquote(text)); print("decoded → clipboard")
    elif a.action == "strip": write(text.strip()); print("stripped → clipboard")
    elif a.action == "lower": write(text.lower()); print("lower → clipboard")
    elif a.action == "upper": write(text.upper()); print("upper → clipboard")
```
Use:
```sh
python clipboard.py urlencode
python clipboard.py strip
```

---

## 5) Text filters: slugify, URL-encode, case tools
**What**: Reusable functions for renaming files or generating IDs.  
**Why**: Consistent naming improves search and automation.
```python
# file: textfilters.py
from __future__ import annotations
import re, unicodedata

def slugify(s: str) -> str:
    s = unicodedata.normalize('NFKD', s)
    s = s.encode('ascii','ignore').decode('ascii')
    s = re.sub(r"[^a-zA-Z0-9]+","-", s).strip('-').lower()
    return s or "n"

if __name__ == "__main__":
    import sys
    print(slugify("Hello, Ned's Report (Oct 2025)"))
```
Example: rename all PDFs in a folder using slugified titles.
```python
from pathlib import Path
from textfilters import slugify
for p in Path("in").glob("*.pdf"):
    p.rename(p.with_name(slugify(p.stem) + p.suffix))
```

---

## 6) Tiny launchers and hotkeys
**What**: One command to trigger common automations.  
**Why**: Muscle‑memory speed.
```python
# file: launcher.py
import argparse, subprocess, sys

ap = argparse.ArgumentParser()
ap.add_argument("task", choices=["tidy","watch","copyurl"])
a = ap.parse_args()

if a.task == "tidy":
    subprocess.run([sys.executable, "organizer.py", "--older-than-days","1"])  # batch tidy
elif a.task == "watch":
    subprocess.run([sys.executable, "watcher.py"])  # realtime
elif a.task == "copyurl":
    # copy active Safari URL (best-effort)
    osa = 'tell application "Safari" to get URL of current tab of front window'
    url = subprocess.check_output(["osascript","-e", osa]).decode().strip()
    subprocess.run([sys.executable, "clipboard.py", "strip"], input=url.encode())
    print(url)
```
Bind to a key with tools like **Raycast**, **Alfred**, or a shell alias.

---

## 7) Scheduling jobs (launchd, cron, Python)
**What**: Run scripts on timers or at login.  
**Why**: Zero‑touch automation.

### 7.1 macOS launchd (recommended on Mac)
Create `~/Library/LaunchAgents/com.ned.automation.daily.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>com.ned.automation.daily</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/env</string><string>bash</string><string>-lc</string>
    <string>source ~/.zprofile && cd ~/Code_Stuff/automation && .venv/bin/python jobs/daily_tidy.py</string>
  </array>
  <key>StartCalendarInterval</key><dict><key>Hour</key><integer>18</integer><key>Minute</key><integer>0</integer></dict>
  <key>StandardOutPath</key><string>~/Library/Logs/automation.daily.log</string>
  <key>StandardErrorPath</key><string>~/Library/Logs/automation.daily.err</string>
  <key>RunAtLoad</key><true/>
</dict>
</plist>
```
Load:
```sh
launchctl load ~/Library/LaunchAgents/com.ned.automation.daily.plist
launchctl start com.ned.automation.daily
```

### 7.2 Cron (portable)
```sh
(crontab -l; echo "0 18 * * * cd ~/Code_Stuff/automation && ~/.pyenv/shims/python jobs/daily_tidy.py") | crontab -
```

### 7.3 Python `schedule` loop (simple)
```sh
pip install schedule
```
```python
import schedule, time, subprocess, sys
schedule.every().day.at("18:00").do(lambda: subprocess.run([sys.executable,"jobs/daily_tidy.py"]))
while True:
    schedule.run_pending(); time.sleep(5)
```

---

## 8) Putting it together: daily tidy pipeline
**What**: One script to tidy Downloads, send a summary, and surface any large files.
```python
# file: jobs/daily_tidy.py
from pathlib import Path
import subprocess, json

ROOT = Path.home()/"Code_Stuff/automation"
python = ROOT/".venv/bin/python"

r = subprocess.run([str(python), str(ROOT/"organizer.py"), "--older-than-days","1"], capture_output=True, text=True)
log = r.stdout

# find large files left in Downloads
leftovers = []
for p in (Path.home()/"Downloads").glob("*"):
    if p.is_file() and p.stat().st_size > 200*1024*1024:
        leftovers.append({"name": p.name, "mb": round(p.stat().st_size/1_000_000,1)})

summary = {
  "moved_log": log.splitlines()[-5:],
  "large_leftovers": leftovers[:10]
}
print(json.dumps(summary, indent=2))

# optional: notify (uncomment)
# subprocess.run([str(python), str(ROOT/"remind.py"), json.dumps(summary), "--title","Daily tidy"])  
```

---

## 9) Troubleshooting
- **Permissions**: Organizer cannot move from protected folders. Grant Terminal Full Disk Access or operate within `~/Downloads`.  
- **Watcher high CPU**: Use non‑recursive watch and debounce writes.  
- **Clipboard empty**: Some apps restrict access; test with `pbcopy/pbpaste`.  
- **launchd not running**: `launchctl list | grep automation`; check logs in `~/Library/Logs`.  
- **Filename collisions**: Code adds numeric suffix; verify target folder if collisions are frequent.

---

## 10) Recap
```plaintext
Define structure → organizer moves files → watchdog reacts to new items → reminders nudge you → clipboard/text filters speed edits → schedule with launchd/cron → compose into daily jobs
```

**Next**: Push logs to chat with `031_NOTIFICATION_AUTOMATION.md`, visualize tidy stats in `011_GRAPHS.md`, and package utilities as a CLI using `035_SCRIPT_PACKAGING.md`. 