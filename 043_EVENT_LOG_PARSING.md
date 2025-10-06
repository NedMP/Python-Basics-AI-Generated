

# Event Log Parsing for Auditing, Alerting, and Analytics — updated Oct 6, 2025

Read and parse **syslog**, **Windows Event Logs**, and **custom app logs** with Python. Extract fields, normalize timestamps, filter by severity, and trigger alerts or generate metrics. Works stand‑alone or as part of a pipeline that ships to email, chat, or a SIEM. Pair with `002_SETUP.md` for environment setup and `016_LOGGING.md` for structured logs.

---

**Assumptions**  
- macOS/Linux for syslog files or `journalctl`; Windows for Event Logs.  
- Python 3.12–3.13 inside a venv.  
- Output targets may include CSV, JSONL, SQLite, or webhooks.

**Install**
```sh
pip install python-dateutil pytz python-dotenv watchdog pywin32  # pywin32 only needed on Windows
```
Optional:
```sh
pip install pandas requests
```

---

## Table of Contents
- [0) Concepts and formats](#0-concepts-and-formats)
- [1) Syslog files (RFC3164/RFC5424)](#1-syslog-files-rfc3164rfc5424)
- [2) systemd-journald via journalctl](#2-systemd-journald-via-journalctl)
- [3) Windows Event Logs (Security/System/Application)](#3-windows-event-logs-securitysystemapplication)
- [4) Custom app logs (text, CSV, JSONL)](#4-custom-app-logs-text-csv-jsonl)
- [5) Tailing, rotation, and state](#5-tailing-rotation-and-state)
- [6) Normalizing timezones and parsing timestamps](#6-normalizing-timezones-and-parsing-timestamps)
- [7) Filtering, severity, and enrichment](#7-filtering-severity-and-enrichment)
- [8) Output: CSV, JSONL, SQLite, and alerts](#8-output-csv-jsonl-sqlite-and-alerts)
- [9) Putting it together: a mini log pipeline](#9-putting-it-together-a-mini-log-pipeline)
- [10) Troubleshooting](#10-troubleshooting)
- [11) Recap](#11-recap)

---

## 0) Concepts and formats
**What**: Logs vary: classic **syslog** (RFC3164), structured **RFC5424**, Windows **Event Logs**, and app logs (text/JSON).  
**Why**: You must parse into a consistent shape: `{ts, host, source, severity, event_id, message, fields...}` for analysis and alerting.

Terms:
- **Facility/Severity** (syslog): e.g., `authpriv.notice`, `daemon.err`.  
- **Event ID** (Windows): numeric identifiers per provider (e.g., 4624 logon).  
- **Rotation**: logs roll to `.1`, `.gz`, or journal; you need resilient tailing.

---

## 1) Syslog files (RFC3164/RFC5424)
**What**: Parse `/var/log/system.log`, `/var/log/syslog`, or app logs written via syslog.  
**Why**: Authentication and daemon events are common audit sources.

### 1.1 Sample lines
```
<34>Oct  5 21:17:03 macbook sshd[123]: Accepted publickey for ned from 1.2.3.4 port 59912 ssh2
<165>1 2025-10-05T21:17:03.312Z host app - ID47 [example@32473 iut="3" eventSource="app"] Started
```

### 1.2 Parser
```python
# file: parse_syslog.py
from __future__ import annotations
import re, socket
from datetime import datetime
from dateutil import tz

RFC3164 = re.compile(r"^<(?P<pri>\d+)>(?P<ts>[A-Z][a-z]{2}\s+\d+\s\d\d:\d\d:\d\d)\s(?P<host>\S+)\s(?P<msg>.*)")
RFC5424  = re.compile(r"^<(?P<pri>\d+)>(?P<ver>\d)\s(?P<ts>\S+)\s(?P<host>\S+)\s(?P<app>\S+)\s(?P<proc>-|\S+)\s(?P<msgid>-|\S+)\s(?P<sd>(\[[^]]+\])+)\s(?P<msg>.*)")

FAC = ["kern","user","mail","daemon","auth","syslog","lpr","news","uucp","cron","authpriv","ftp"] + [f"local{i}" for i in range(8)]
SEV = ["emerg","alert","crit","err","warning","notice","info","debug"]

THIS_HOST = socket.gethostname()

def parse(line: str) -> dict | None:
    m = RFC5424.match(line)
    if m:
        pri = int(m['pri']); facility = FAC[(pri>>3) % len(FAC)]; severity = SEV[pri % 8]
        ts = m['ts']
        dt = datetime.fromisoformat(ts.replace('Z','+00:00'))
        return {"ts": dt.astimezone(tz.UTC).isoformat(), "host": m['host'], "facility": facility, "severity": severity, "app": m['app'], "msg": m['msg']}
    m = RFC3164.match(line)
    if m:
        pri = int(m['pri']); facility = FAC[(pri>>3) % len(FAC)]; severity = SEV[pri % 8]
        # RFC3164 lacks year; assume current year and local tz
        dt = datetime.strptime(m['ts'], "%b %d %H:%M:%S").replace(year=datetime.now().year)
        return {"ts": dt.astimezone().astimezone(tz.UTC).isoformat(), "host": m['host'], "facility": facility, "severity": severity, "app": None, "msg": m['msg']}
    return None
```

Run on a file:
```python
for line in open('/var/log/system.log','r',encoding='utf-8', errors='ignore'):
    evt = parse(line)
    if evt and evt['severity'] in {'err','crit','alert','emerg'}:
        print(evt)
```

---

## 2) systemd-journald via journalctl
**What**: Read logs from the binary journal on Linux.  
**Why**: Central source for units and kernel with structured fields.
```python
# file: parse_journal.py
import json, subprocess
# last 100 auth events in JSON
p = subprocess.run(['journalctl','-u','sshd','-n','100','-o','json'], capture_output=True, text=True, check=True)
for line in p.stdout.splitlines():
    rec = json.loads(line)
    evt = {
        'ts': rec.get('__REALTIME_TIMESTAMP'),
        'host': rec.get('_HOSTNAME'),
        'unit': rec.get('_SYSTEMD_UNIT'),
        'msg': rec.get('MESSAGE')
    }
    print(evt)
```
Tail mode:
```sh
journalctl -f -o json
```
Parse lines as above.

---

## 3) Windows Event Logs (Security/System/Application)
**What**: Query Event Logs by channel and filter for event IDs.  
**Why**: Audit logons, service failures, policy changes.
```python
# file: win_events.py  (Windows only)
import win32evtlog, win32evtlogutil, win32con
import xml.etree.ElementTree as ET

server = 'localhost'; logtype = 'Security'  # 'System','Application'
h = win32evtlog.EvtQuery(f"{logtype}", win32evtlog.EvtQueryReverseDirection, "Event[System[EventID=4625]]")
while True:
    events = win32evtlog.EvtNext(h, 16)
    if not events: break
    for e in events:
        xml = win32evtlog.EvtRender(e, win32evtlog.EvtRenderEventXml)
        root = ET.fromstring(xml)
        sys = root.find('System')
        ts = sys.find('TimeCreated').attrib.get('SystemTime')
        eid = int(sys.find('EventID').text)
        provider = sys.find('Provider').attrib.get('Name')
        msg = root.find('EventData')
        fields = {d.attrib.get('Name'): (d.text or '') for d in msg} if msg is not None else {}
        print({'ts': ts, 'event_id': eid, 'provider': provider, 'fields': fields})
```
Notes:
- Common IDs: **4624** (logon success), **4625** (logon failure), **4720** (user created), **6006** (event log service stopped).  
- Filter XPath to your needs (e.g., date ranges or specific accounts).

---

## 4) Custom app logs (text, CSV, JSONL)
**What**: Parse arbitrary patterns or structured lines.  
**Why**: Many apps write plain text or JSON per line.

### 4.1 Regex text parser
```python
import re
LINE = re.compile(r"^(?P<ts>\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}Z)\s+(?P<level>INFO|WARN|ERROR)\s+(?P<msg>.*)$")
```

### 4.2 JSONL
```python
import json
for line in open('app.log.jsonl','r',encoding='utf-8'):
    rec = json.loads(line)
    if rec.get('level') in {'ERROR','WARN'}:
        ...
```

### 4.3 CSV
```python
import csv
with open('events.csv', newline='') as f:
    for row in csv.DictReader(f):
        ...
```

---

## 5) Tailing, rotation, and state
**What**: Follow a file like `tail -f`, survive rotations, and avoid duplicates.  
**Why**: Production logs roll frequently.
```python
# file: tail.py
from __future__ import annotations
from pathlib import Path
import os, time

def tail_follow(path: Path, state: Path):
    off = 0
    if state.exists(): off = int(state.read_text())
    with path.open('rb') as f:
        f.seek(off)
        inode = os.fstat(f.fileno()).st_ino
        while True:
            where = f.tell()
            line = f.readline()
            if not line:
                time.sleep(0.5)
                # rotated?
                try:
                    if os.stat(path).st_ino != inode:
                        f.close(); f = path.open('rb'); inode = os.fstat(f.fileno()).st_ino
                except FileNotFoundError:
                    time.sleep(1)
                continue
            yield line.decode('utf-8', 'ignore').rstrip('\n')
            state.write_text(str(f.tell()))
```
Usage:
```python
for ln in tail_follow(Path('/var/log/system.log'), Path('.state.offset')):
    print(ln)
```

---

## 6) Normalizing timezones and parsing timestamps
**What**: Bring all timestamps to **UTC ISO‑8601**.  
**Why**: Cross‑system correlation and correct windowing.
```python
from dateutil import parser, tz

def to_utc_iso(s: str) -> str:
    dt = parser.parse(s)
    if not dt.tzinfo:
        dt = dt.replace(tzinfo=tz.tzlocal())
    return dt.astimezone(tz.UTC).isoformat()
```

---

## 7) Filtering, severity, and enrichment
**What**: Drop noise and add context fields.  
**Why**: Higher signal, better alerts.
- Map severities to ints: `ERROR=3, WARN=4, INFO=6`.  
- Add `host`, `app`, and `env` from config.  
- Derive fields: IPs (`re.findall(r"\b\d+\.\d+\.\d+\.\d+\b", msg)`), usernames, or device IDs.

---

## 8) Output: CSV, JSONL, SQLite, and alerts
**What**: Store or forward normalized events and emit alerts on rules.  
**Why**: Persist for analytics and notify quickly.
```python
# CSV
import csv
with open('out/events.csv','a',newline='') as f:
    w = csv.DictWriter(f, fieldnames=['ts','host','source','severity','event_id','message'])
    if f.tell()==0: w.writeheader()
    w.writerow(evt)
```
```python
# JSONL
import json
open('out/events.jsonl','a').write(json.dumps(evt)+"\n")
```
```python
# SQLite (lightweight analytics)
import sqlite3
con = sqlite3.connect('out/events.db')
con.execute('CREATE TABLE IF NOT EXISTS events(ts TEXT, host TEXT, source TEXT, severity TEXT, event_id TEXT, message TEXT)')
con.execute('INSERT INTO events VALUES (?,?,?,?,?,?)', (evt['ts'], evt['host'], evt.get('source','syslog'), evt['severity'], str(evt.get('event_id','')), evt['msg']))
con.commit()
```
Alerts: integrate with `031_NOTIFICATION_AUTOMATION.md` to post to Slack/Teams/email on conditions like repeated failures or brute‑force patterns.

---

## 9) Putting it together: a mini log pipeline
**What**: End‑to‑end script that tails syslog, normalizes, writes JSONL, and alerts on auth failures.
```python
# file: pipeline.py
from pathlib import Path
import re, json
from parse_syslog import parse
from tail import tail_follow

FAIL = re.compile(r"Failed password for (?P<user>\S+) from (?P<ip>\d+\.\d+\.\d+\.\d+)")

for line in tail_follow(Path('/var/log/system.log'), Path('.state.offset')):
    evt = parse(line)
    if not evt: continue
    evt['source'] = 'syslog'
    open('out/events.jsonl','a').write(json.dumps(evt)+"\n")
    m = FAIL.search(evt['msg'])
    if m:
        print('ALERT', m.groupdict())  # replace with Slack/Email sender
```

---

## 10) Troubleshooting
- **Permissions**: reading system logs may require sudo/admin or membership in a log group.  
- **Rotation gaps**: store offsets and handle inode changes (see tailing code).  
- **Timestamp parse failures**: add explicit formatters for odd formats.  
- **Windows `pywin32` import error**: install wheels that match Python version and architecture.  
- **High CPU in tail**: increase sleep interval and debounce duplicate lines.

---

## 11) Recap
```plaintext
Identify source → parse lines → normalize timestamps → filter/enrich → store (CSV/JSONL/DB) → alert on rules → schedule + monitor
```

**Next**: Visualize counts in `011_GRAPHS.md`, package as a CLI (`035_SCRIPT_PACKAGING.md`), schedule collection (`026_TASK_SCHEDULING.md`), and secure credentials (`033_KEYCHAIN_SECRETS.md`).