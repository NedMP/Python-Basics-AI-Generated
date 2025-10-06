

# Lightweight Monitoring Scripts with Python — updated Oct 6, 2025

Practical patterns for building **small, reliable monitors** for hosts, services, and logs. Covers health checks (ICMP/TCP/HTTP), system metrics, log tailing, simple state tracking, and sending alerts via **email**, **Slack**, or **Microsoft Teams**. Pairs with `016_LOGGING.md`, `020_API_AUTOMATION.md`, `014_EMAILS.md`, and scheduling in `026_TASK_SCHEDULING.md`.

---

**Assumptions**  
- macOS or Linux shell, Python 3.12–3.13 in a venv.  
- Config from `.env` and/or a small YAML/TOML file.  
- You prefer **simple scripts** over heavy monitoring stacks.

**Install**
```sh
pip install requests python-dotenv pyyaml tomlkit psutil "urllib3>=2"
```

---

## Table of Contents
- [0) Monitoring philosophy](#0-monitoring-philosophy)
- [1) Config and secrets](#1-config-and-secrets)
- [2) Connectivity checks: ping, TCP, HTTP](#2-connectivity-checks-ping-tcp-http)
- [3) System metrics: CPU, memory, disk, processes](#3-system-metrics-cpu-memory-disk-processes)
- [4) Log monitoring and patterns](#4-log-monitoring-and-patterns)
- [5) Service checks via subprocess](#5-service-checks-via-subprocess)
- [6) Alerts: email, Slack, Teams](#6-alerts-email-slack-teams)
- [7) State, deduplication, and flapping control](#7-state-deduplication-and-flapping-control)
- [8) Scheduling and runtime](#8-scheduling-and-runtime)
- [9) Packaging and structure](#9-packaging-and-structure)
- [10) Testing and dry runs](#10-testing-and-dry-runs)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Monitoring philosophy
**What**: Small scripts that answer a **yes/no** or produce a **few numbers**.  
**Why**: Fast to deploy, easy to reason about, good for gaps your main platform doesn’t cover.
- Prefer idempotent checks that complete in seconds.  
- Emit structured logs (`016_LOGGING.md`) and short alert text.  
- Fail closed: if a check can’t run, alert with context.

---

## 1) Config and secrets
**What**: Keep behavior out of code.  
**Why**: Change targets without redeploying.

`.env` (for secrets):
```
SLACK_WEBHOOK=https://hooks.slack.com/services/XXX/YYY/ZZZ
SMTP_HOST=smtp.example.com
SMTP_USER=alerts@example.com
SMTP_PASS=... 
```
`config.yaml` (for targets):
```yaml
http_checks:
  - name: portal
    url: https://example.com/health
    timeout: 5
    expect_status: 200
    expect_text: ok
hosts:
  - 1.1.1.1
  - 8.8.8.8
```
Loader:
```python
# file: cfg.py
from dotenv import load_dotenv; load_dotenv()
import yaml, os

def load_config(path="config.yaml"):
    with open(path, "r", encoding="utf-8") as f:
        return yaml.safe_load(f) or {}
```

---

## 2) Connectivity checks: ping, TCP, HTTP
**What**: Basic reachability and service health.

### 2.1 Ping (system ping, portable)
```python
import subprocess, shlex

def ping(host: str, count=2, timeout=2) -> bool:
    cmd = f"ping -c {count} -W {timeout} {shlex.quote(host)}"
    return subprocess.run(cmd, shell=True).returncode == 0
```

### 2.2 TCP port
```python
import socket

def tcp_open(host: str, port: int, timeout=2.0) -> bool:
    try:
        with socket.create_connection((host, port), timeout=timeout):
            return True
    except OSError:
        return False
```

### 2.3 HTTP health
```python
import requests

def http_ok(url: str, timeout=5, expect_status=200, expect_text=None) -> tuple[bool,str]:
    r = requests.get(url, timeout=timeout)
    ok = (r.status_code == expect_status) and ((expect_text is None) or (expect_text in r.text))
    return ok, f"{r.status_code} in {r.elapsed.total_seconds():.2f}s"
```

---

## 3) System metrics: CPU, memory, disk, processes
**What**: Threshold triggers for host health.
```python
import psutil

def metrics():
    return {
        "cpu": psutil.cpu_percent(interval=0.5),
        "mem": psutil.virtual_memory().percent,
        "disk": psutil.disk_usage("/").percent,
        "procs": len(psutil.pids()),
    }

m = metrics()
if m["cpu"] > 90 or m["mem"] > 90 or m["disk"] > 90:
    print("THRESHOLD BREACH", m)
```

---

## 4) Log monitoring and patterns
**What**: Tail files for error patterns and volume spikes.
```python
# file: log_watch.py
from pathlib import Path
import time, re

pattern = re.compile(r"ERROR|CRITICAL|traceback", re.I)

def tail(path: Path):
    with path.open("r", encoding="utf-8", errors="ignore") as f:
        f.seek(0, 2)
        while True:
            line = f.readline()
            if not line:
                time.sleep(0.5); continue
            if pattern.search(line):
                yield line.strip()
```
Use log rotation friendly libraries if needed (`watchdog`, or read from journald/syslog via `subprocess`).

---

## 5) Service checks via subprocess
**What**: Verify local services quickly.
```python
import subprocess

def systemd_active(unit: str, timeout=5) -> bool:
    try:
        subprocess.run(["systemctl", "is-active", "--quiet", unit], timeout=timeout, check=True)
        return True
    except subprocess.CalledProcessError:
        return False
```
macOS launchd: use `launchctl print system/<label>` and parse.

---

## 6) Alerts: email, Slack, Teams
**What**: Send compact alerts with context.

### 6.1 Email (SMTP)
```python
import os, smtplib
from email.message import EmailMessage

def send_email(subject: str, body: str, to: list[str]):
    msg = EmailMessage()
    msg["From"] = os.getenv("SMTP_USER")
    msg["To"] = ", ".join(to)
    msg["Subject"] = subject
    msg.set_content(body)
    with smtplib.SMTP(os.getenv("SMTP_HOST"), 587) as s:
        s.starttls()
        s.login(os.getenv("SMTP_USER"), os.getenv("SMTP_PASS"))
        s.send_message(msg)
```

### 6.2 Slack Incoming Webhook
```python
import os, requests

def slack(text: str):
    url = os.getenv("SLACK_WEBHOOK")
    requests.post(url, json={"text": text}, timeout=5)
```

### 6.3 Microsoft Teams Webhook
```python
import os, requests

def teams(text: str):
    url = os.getenv("TEAMS_WEBHOOK")
    card = {"@type":"MessageCard","@context":"https://schema.org/extensions","text":text}
    requests.post(url, json=card, timeout=5)
```

**Alert text**: include check name, target, failure detail, and a short hint. Keep under 500 chars.

---

## 7) State, deduplication, and flapping control
**What**: Avoid noisy repeats.
```python
from pathlib import Path
import json, time

STATE = Path(".state.json")

def should_alert(key: str, ok: bool, cool=300) -> bool:
    state = json.loads(STATE.read_text()) if STATE.exists() else {}
    last = state.get(key, {"ok": True, "ts": 0})
    now = time.time()
    fire = (ok is False and (last["ok"] is True or now - last["ts"] > cool))
    state[key] = {"ok": ok, "ts": now}
    STATE.write_text(json.dumps(state))
    return fire

# usage
ok, info = http_ok("https://example.com/health")
if should_alert("http:example", ok):
    slack(f"HTTP FAIL example.com → {info}")
```
Notes:
- Use **cooldowns** to suppress repeats.  
- Consider **two‑strike** policy before first alert.  
- Reset state on success to allow future alerts.

---

## 8) Scheduling and runtime
**What**: Run periodically.
- Local dev: `watch -n 60 python check.py` or `while true; do ...; sleep 60; done`.  
- Production: cron, `launchd` (macOS), or see `026_TASK_SCHEDULING.md` for `APScheduler` and reliability tips.  
- Use **timeouts** on every network call. Log every run at INFO level.

---

## 9) Packaging and structure
**What**: Keep scripts maintainable.
```
monitoring/
├── check_http.py
├── check_ping.py
├── log_watch.py
├── config.yaml
├── .env
└── utils/
    ├── cfg.py
    ├── alerts.py
    └── state.py
```
Guidelines:
- One check per file. Keep functions pure; return `(ok, info)` tuples.  
- Reuse `alerts.py` and `state.py`.  
- Add `requirements.txt` and pin versions for reproducibility.

---

## 10) Testing and dry runs
**What**: Validate without spamming real channels.
- Add `--dry-run` to print would‑send messages.  
- Unit test check functions by mocking network/filesystem.  
- Capture logs as described in `016_LOGGING.md` Section 10.

---

## 11) Troubleshooting
- **TLS/SSL errors**: update `certifi`; avoid `verify=False`.  
- **ICMP requires root** on macOS: call system `ping` via subprocess.  
- **Webhook 400/403**: URL wrong or body format invalid. Recreate webhook.  
- **Cron PATH differences**: set absolute paths and `PATH` inside the crontab.  
- **Log rotation**: reopen files or use inode-aware tail logic.  
- **Noisy flapping**: raise thresholds, add two‑strike logic, extend cooldown.

---

## 12) Recap
```plaintext
Load config + .env → run fast checks (ping/TCP/HTTP/metrics/logs) → compute (ok, info) → suppress duplicates with state → alert via email/Slack/Teams → schedule with cron/APScheduler → log outcomes and tune thresholds
```

**Next**: Automate schedules in `026_TASK_SCHEDULING.md`, format alerts with `016_LOGGING.md` patterns, and use `014_EMAILS.md` for richer email templates or enterprise Graph auth.