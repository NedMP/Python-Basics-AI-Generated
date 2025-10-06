

# Scheduling Python Tasks — updated Oct 6, 2025

Practical guide to running Python scripts on a schedule using **`schedule`**, **APScheduler**, classic **cron**, and macOS **launchd**. Covers when to choose each tool, environment handling, logging, retries, health checks, and safe shutdown. Use with `002_SETUP.md` for venv and `.env` management and `016_LOGGING.md` for structured logs.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 inside a venv.  
- Jobs are idempotent and finish within their interval unless stated otherwise.

**Install**
```sh
pip install schedule apscheduler python-dotenv
```

---

## Table of Contents
- [0) Choosing a scheduler](#0-choosing-a-scheduler)
- [1) One‑file jobs with `schedule`](#1-one-file-jobs-with-schedule)
- [2) APScheduler for robust in‑process scheduling](#2-apscheduler-for-robust-in-process-scheduling)
- [3) System cron](#3-system-cron)
- [4) macOS launchd](#4-macos-launchd)
- [5) Logging, retries, and notifications](#5-logging-retries-and-notifications)
- [6) Environment, paths, and concurrency](#6-environment-paths-and-concurrency)
- [7) Testing and dry runs](#7-testing-and-dry-runs)
- [8) Troubleshooting](#8-troubleshooting)
- [9) Recap](#9-recap)

---

## 0) Choosing a scheduler
**What**: Pick the simplest tool that meets requirements.  
**Why**: Reduces failure modes and maintenance.
- **`schedule`**: tiny, single‑process loop for one script. No persistence. Good for desktop utilities or short‑lived jobs.  
- **APScheduler**: rich triggers (cron/interval/date), job stores, misfire handling, concurrency control. Run inside your app or a small worker service.  
- **cron**: OS native, minimal overhead. Best for CLI scripts that exit. Logs and env must be set carefully.  
- **launchd (macOS)**: per‑user or system agents with start intervals, calendars, and KeepAlive. Replaces cron on macOS by default.

---

## 1) One‑file jobs with `schedule`
**What**: Minimal scheduler that runs functions at intervals inside a while‑loop.  
**Why**: Quick wins with few dependencies.
```python
# file: job_schedule.py
import os, time
from dotenv import load_dotenv
import schedule

load_dotenv()

def job():
    name = os.getenv("ENV_NAME", "World")
    print("running job for", name)

schedule.every(1).minutes.do(job)        # or: .hour, .day, .monday, etc.
# schedule.every().day.at("09:30").do(job)

if __name__ == "__main__":
    while True:
        schedule.run_pending()
        time.sleep(1)
```
Run:
```sh
python job_schedule.py
```
Notes:
- Process must stay alive. Use a supervisor (launchd/pm2) if you need auto‑restart.  
- No built‑in parallelism; ensure `job()` completes quickly.

---

## 2) APScheduler for robust in‑process scheduling
**What**: Enterprise‑friendly scheduling inside your Python service.  
**Why**: Cron/interval triggers, job persistence, overlapping control, and misfire handling.

### 2.1 Basic interval/cron jobs
```python
# file: apsched_basic.py
from datetime import datetime
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger

sched = BackgroundScheduler()

# every 5 minutes
sched.add_job(lambda: print("tick", datetime.now()), "interval", minutes=5, id="tick5")

# 9:30 on weekdays Sydney time
sched.add_job(lambda: print("report"), CronTrigger.from_crontab("30 9 * * 1-5", timezone="Australia/Melbourne"), id="daily_report")

sched.start()

try:
    import time
    while True:
        time.sleep(1)
except KeyboardInterrupt:
    sched.shutdown(wait=True)
```

### 2.2 Job stores and executors
Persist jobs and run with thread or process pools.
```python
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.jobstores.sqlalchemy import SQLAlchemyJobStore
from apscheduler.executors.pool import ThreadPoolExecutor, ProcessPoolExecutor

jobstores = {"default": SQLAlchemyJobStore(url="sqlite:///jobs.sqlite")}
executors = {"default": ThreadPoolExecutor(10), "processpool": ProcessPoolExecutor(2)}
job_defaults = {"coalesce": True, "max_instances": 1, "misfire_grace_time": 60}

sched = BackgroundScheduler(jobstores=jobstores, executors=executors, job_defaults=job_defaults, timezone="Australia/Melbourne")
```
Notes:
- `coalesce=True` merges missed runs into one.  
- `max_instances=1` prevents overlaps for the same job.

---

## 3) System cron
**What**: Run scripts at fixed times using the OS.  
**Why**: No always‑running Python process required.

### 3.1 Create a crontab entry
```sh
crontab -e
```
Example (every 5 minutes):
```
*/5 * * * * /bin/zsh -lc 'cd /Users/ned/Code_Stuff/hello-venv && source .venv/bin/activate && python check_http.py >> logs/check_http.log 2>&1'
```
Notes:
- Cron has a minimal environment. Export `PATH` and use absolute paths.  
- Redirect stdout/stderr to log files for diagnostics.  
- Use `-l` with zsh to load your login environment, or set variables explicitly.

### 3.2 Debugging cron
Check logs:
```sh
grep CRON /var/log/system.log 2>/dev/null || echo "Use launchd logs on new macOS"
```
On macOS, prefer launchd for better logging.

---

## 4) macOS launchd
**What**: Native scheduler and supervisor for macOS.  
**Why**: Better than cron on macOS; supports KeepAlive and calendar intervals.

### 4.1 Per‑user agent (recommended)
Create `~/Library/LaunchAgents/com.ned.python.check.plist`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.ned.python.check</string>
  <key>ProgramArguments</key>
  <array>
    <string>/bin/zsh</string>
    <string>-lc</string>
    <string>cd /Users/ned/Code_Stuff/hello-venv && source .venv/bin/activate && python check_http.py</string>
  </array>
  <key>StartInterval</key><integer>300</integer>
  <key>StandardOutPath</key><string>/Users/ned/Code_Stuff/hello-venv/logs/check_http.out</string>
  <key>StandardErrorPath</key><string>/Users/ned/Code_Stuff/hello-venv/logs/check_http.err</string>
  <key>KeepAlive</key><false/>
  <key>EnvironmentVariables</key>
  <dict>
    <key>PYTHONUNBUFFERED</key><string>1</string>
  </dict>
</dict></plist>
```
Load and start:
```sh
launchctl load ~/Library/LaunchAgents/com.ned.python.check.plist
launchctl start com.ned.python.check
```
Unload:
```sh
launchctl unload ~/Library/LaunchAgents/com.ned.python.check.plist
```

### 4.2 Calendar‑based
Replace `StartInterval` with:
```xml
<key>StartCalendarInterval</key>
<dict><key>Hour</key><integer>9</integer><key>Minute</key><integer>30</integer></dict>
```
Multiple schedules are arrays of dicts.

---

## 5) Logging, retries, and notifications
**What**: Make failures actionable and non‑noisy.  
**Why**: You need context to fix problems fast.
- Use `logging` per `016_LOGGING.md`, write to both console and rotating files.  
- Add retries with jitter for transient network errors.  
- Send alerts via Slack/Teams/email (`025_MONITORING_SCRIPTS.md`) on final failure.  
- Include job name, last run time, attempt count, and elapsed seconds in the message.

Example retry wrapper:
```python
import time, random

def retry(fn, attempts=4, base=0.5, factor=2.0):
    for i in range(attempts):
        try:
            return fn()
        except Exception as e:
            if i == attempts - 1:
                raise
            time.sleep(base * (factor ** i) + random.random()/10)
```

---

## 6) Environment, paths, and concurrency
**What**: Most scheduler bugs are environment/path issues or overlapping runs.
- Always use absolute paths for `python`, scripts, venv, and logs.  
- Export required env vars explicitly or use `.env` loader in the script.  
- Prevent overlaps: with APScheduler set `max_instances=1`; with cron/launchd use **lock files**.

Lock file pattern:
```python
from pathlib import Path
import sys
lock = Path("/tmp/myjob.lock")
if lock.exists():
    sys.exit(0)
try:
    lock.touch(exist_ok=False)
    # ... run job ...
finally:
    if lock.exists():
        lock.unlink()
```

---

## 7) Testing and dry runs
**What**: Validate schedules without side effects.
- Add `--dry-run` to jobs to skip writes and external calls.  
- For APScheduler, shorten intervals to seconds in dev.  
- For cron/launchd, write a temporary plist/crontab with a 1‑minute cadence and log to a sandbox path.

---

## 8) Troubleshooting
- **Works in terminal, fails in cron/launchd**: PATH or venv not loaded. Use full paths and `zsh -lc 'source .venv/bin/activate && ...'`.  
- **Multiple overlapping runs**: enforce lock files or APScheduler `max_instances=1`.  
- **Time zone confusion**: set explicit timezone in APScheduler triggers; cron uses system time.  
- **Script prints nothing**: buffer disabled? Set `PYTHONUNBUFFERED=1` or flush logs.  
- **launchd won’t start**: check `launchctl print gui/$(id -u)/com.ned.python.check` and file paths; plist must be valid XML.

---

## 9) Recap
```plaintext
Pick tool → write idempotent job → add logging + retries → schedule with schedule/APScheduler or cron/launchd → use absolute paths and env → prevent overlaps → test with short intervals → monitor logs and alerts
```

**Next**: Wire results into notifications (`031_NOTIFICATION_AUTOMATION.md`) and email reports (`014_EMAILS.md`). For long‑running services, consider a process supervisor (launchd) or containerized cron in CI/CD.