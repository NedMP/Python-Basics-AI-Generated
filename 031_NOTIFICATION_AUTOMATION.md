

# Notification Automation (Slack, Teams, Discord, Email) — updated Oct 6, 2025

Practical patterns for sending **automated notifications** from Python to Slack, Microsoft Teams, Discord, or Email. Covers webhook setup, message formatting (blocks/cards/embeds), attachments, threading, deduplication, **rate control** with retries/backoff, and security. Pair with `025_MONITORING_SCRIPTS.md` for producing alerts and `026_TASK_SCHEDULING.md` for running jobs on a schedule.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Secrets in `.env` loaded via `python-dotenv`.  
- Outbound HTTPS allowed.

**Install**
```sh
pip install requests python-dotenv pydantic "urllib3>=2"
```

---

## Table of Contents
- [0) Choosing a channel](#0-choosing-a-channel)
- [1) Environment and config](#1-environment-and-config)
- [2) Slack webhooks: blocks, mentions, threads](#2-slack-webhooks-blocks-mentions-threads)
- [3) Microsoft Teams webhooks: MessageCard/Adaptive Card](#3-microsoft-teams-webhooks-messagecardadaptive-card)
- [4) Discord webhooks: embeds and files](#4-discord-webhooks-embeds-and-files)
- [5) Email (SMTP) fallback](#5-email-smtp-fallback)
- [6) Rate limits, retries, and backoff](#6-rate-limits-retries-and-backoff)
- [7) De-duplication and suppression](#7-de-duplication-and-suppression)
- [8) Attachments, threading, and formatting tips](#8-attachments-threading-and-formatting-tips)
- [9) Validation with Pydantic](#9-validation-with-pydantic)
- [10) Small notifier module](#10-small-notifier-module)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Choosing a channel
**What**: Pick the destination based on urgency, audience, and auditability.  
**Why**: Right channel reduces noise and increases action.
- **Slack**: team ops, rich formatting, threads.  
- **Teams**: enterprise first; use MessageCard or Adaptive Card.  
- **Discord**: hobby projects, embeds, quick webhooks.  
- **Email**: long form, attachments, archival.

---

## 1) Environment and config
**What**: Centralize secrets and destinations.  
**Why**: Keep code portable and safe.

`.env` example:
```
SLACK_WEBHOOK=https://hooks.slack.com/services/XXX/YYY/ZZZ
TEAMS_WEBHOOK=https://outlook.office.com/webhook/...
DISCORD_WEBHOOK=https://discord.com/api/webhooks/...
SMTP_HOST=smtp.example.com
SMTP_USER=alerts@example.com
SMTP_PASS=...
ALERT_CHANNEL=slack   # slack|teams|discord|email
```
Loader:
```python
from dotenv import load_dotenv; load_dotenv()
import os
DEST = os.getenv("ALERT_CHANNEL", "slack")
```

---

## 2) Slack webhooks: blocks, mentions, threads
**What**: Send structured messages with Blocks; mention users or channels.
```python
import os, requests

SLACK_WEBHOOK = os.getenv("SLACK_WEBHOOK")

def slack_text(text: str):
    r = requests.post(SLACK_WEBHOOK, json={"text": text}, timeout=5)
    r.raise_for_status()

# Blocks with fields and code block
payload = {
  "blocks": [
    {"type":"section","text":{"type":"mrkdwn","text":"*Deployment succeeded* :white_check_mark:"}},
    {"type":"section","fields":[
      {"type":"mrkdwn","text":"*Service*\napi-gateway"},
      {"type":"mrkdwn","text":"*Duration*\n42s"}
    ]},
    {"type":"section","text":{"type":"mrkdwn","text":"```commit: 1a2b3c\nby: ned\n```"}}
  ]
}
requests.post(SLACK_WEBHOOK, json=payload, timeout=5).raise_for_status()
```
Mentions:
- User: `<@U123456>`  
- Channel: `<!channel>` or `<!here>` (use sparingly).

Threading requires a bot token and Web API; webhooks alone cannot start a new thread. For simple webhooks, include a correlation ID to group messages.

---

## 3) Microsoft Teams webhooks: MessageCard/Adaptive Card
**What**: Post compact alerts into a Teams channel.
```python
import os, requests
TEAMS_WEBHOOK = os.getenv("TEAMS_WEBHOOK")
card = {
  "@type":"MessageCard","@context":"https://schema.org/extensions",
  "summary":"Health check failed",
  "themeColor":"E81123",
  "title":"HTTP check",
  "sections":[{"facts":[{"name":"Target","value":"example.com"},{"name":"Status","value":"503"}]}],
  "potentialAction":[{"@type":"OpenUri","name":"Open dashboard","targets":[{"os":"default","uri":"https://status.example.com"}]}]
}
requests.post(TEAMS_WEBHOOK, json=card, timeout=5).raise_for_status()
```
Note: Some tenants prefer **Incoming Webhook (MessageCard)**, others require **Power Automate** or **Adaptive Cards** via bot—adjust to policy.

---

## 4) Discord webhooks: embeds and files
**What**: Quick alerts with rich embeds and file uploads.
```python
import os, requests
DISCORD_WEBHOOK = os.getenv("DISCORD_WEBHOOK")

embed = {
  "title": "Backup completed",
  "description": "Daily snapshot archived",
  "color": 0x2ecc71,
  "fields": [
    {"name":"Archive","value":"projects_20251006-120000.tar.gz","inline":True},
    {"name":"Size","value":"1.4 GB","inline":True}
  ]
}
requests.post(DISCORD_WEBHOOK, json={"embeds":[embed]}, timeout=5).raise_for_status()

# Upload a small text file
files = {"file": ("log.txt", b"ok\n", "text/plain")}
requests.post(DISCORD_WEBHOOK, files=files, timeout=5).raise_for_status()
```

---

## 5) Email (SMTP) fallback
**What**: Use for longer reports or when chat apps are blocked.
```python
import os, smtplib
from email.message import EmailMessage

SMTP_HOST=os.getenv("SMTP_HOST"); SMTP_USER=os.getenv("SMTP_USER"); SMTP_PASS=os.getenv("SMTP_PASS")

def send_email(subject: str, body: str, to: list[str]):
    msg = EmailMessage(); msg["From"] = SMTP_USER; msg["To"] = ", ".join(to); msg["Subject"] = subject
    msg.set_content(body)
    with smtplib.SMTP(SMTP_HOST, 587) as s:
        s.starttls(); s.login(SMTP_USER, SMTP_PASS); s.send_message(msg)
```
Tip: Prefer transactional providers (Graph API, SES, SendGrid) in production for deliverability and rate control.

---

## 6) Rate limits, retries, and backoff
**What**: Avoid bans and deliver messages reliably.  
**Why**: Webhooks return **429** when throttled; transient 5xx happens.
```python
import time, random, requests

def post_with_retry(url: str, json=None, files=None, attempts=5, base=0.5):
    for i in range(attempts):
        r = requests.post(url, json=json, files=files, timeout=10)
        if r.status_code in (200, 204):
            return r
        if r.status_code == 429:
            wait = float(r.headers.get("Retry-After", base*(2**i)+random.random()/10))
            time.sleep(wait); continue
        if 500 <= r.status_code < 600:
            time.sleep(base*(2**i)+random.random()/10); continue
        r.raise_for_status()
    r.raise_for_status()
```
Controls:
- Backoff with jitter, honor `Retry-After`.  
- Limit concurrency.  
- Enforce per‑destination QPS.

---

## 7) De-duplication and suppression
**What**: Stop noisy repeats and collapse storms.
```python
from pathlib import Path
import json, time
STATE = Path(".notify_state.json")

def should_send(key: str, cool=300) -> bool:
    st = json.loads(STATE.read_text()) if STATE.exists() else {}
    now = time.time(); last = st.get(key, 0)
    if now - last < cool:
        return False
    st[key] = now; STATE.write_text(json.dumps(st))
    return True

# usage
k = "http:example.com:503"
if should_send(k, cool=600):
    slack_text("HTTP 503 on example.com")
```
Variants: two‑strike before first alert, and success‑resets.

---

## 8) Attachments, threading, and formatting tips
**Slack**
- Use **Blocks**; keep text under 3000 chars per block.  
- For logs, attach files via Slack Web API (requires bot token) or host and link.  
- Include a **correlation ID** and compact JSON code blocks.

**Teams**
- Keep MessageCard under size limits; prefer links to large payloads.  
- Use `themeColor` (hex) for severity.

**Discord**
- Max 10 embeds per message; prefer concise fields.  
- Files limited by size; compress logs.

**Email**
- Plain text fallback plus optional HTML.  
- Keep subjects descriptive with a short prefix, e.g., `[ALERT][PROD]`.

---

## 9) Validation with Pydantic
**What**: Catch malformed payloads early in code.
```python
from pydantic import BaseModel, Field, HttpUrl

class SlackBlock(BaseModel):
    type: str = Field("section", pattern=r"^(section|divider|image)$")
    text: dict

class SlackPayload(BaseModel):
    blocks: list[SlackBlock]

SlackPayload(blocks=[{"type":"section","text":{"type":"mrkdwn","text":"ok"}}])
```

---

## 10) Small notifier module
**What**: One interface to multiple channels with rate control.
```python
# file: notifier.py
import os, requests, time, random
from dotenv import load_dotenv
load_dotenv()

SLACK = os.getenv("SLACK_WEBHOOK"); TEAMS = os.getenv("TEAMS_WEBHOOK"); DISCORD = os.getenv("DISCORD_WEBHOOK")

def _post(url, payload=None, files=None, attempts=5, base=0.5):
    for i in range(attempts):
        r = requests.post(url, json=payload, files=files, timeout=10)
        if r.status_code in (200,204): return True
        if r.status_code == 429:
            wait = float(r.headers.get("Retry-After", base*(2**i)))
            time.sleep(wait); continue
        if 500 <= r.status_code < 600:
            time.sleep(base*(2**i)+random.random()/10); continue
        return False
    return False

def send(text: str, channel: str = "slack") -> bool:
    if channel == "slack" and SLACK:
        return _post(SLACK, {"text": text})
    if channel == "teams" and TEAMS:
        card = {"@type":"MessageCard","@context":"https://schema.org/extensions","text": text}
        return _post(TEAMS, card)
    if channel == "discord" and DISCORD:
        return _post(DISCORD, {"content": text})
    return False

if __name__ == "__main__":
    send("Notifier smoke test: ok")
```

---

## 11) Troubleshooting
- **HTTP 429**: back off; confirm org limits and rotate to secondary channel.  
- **400/403**: invalid webhook URL or payload shape; recreate webhook.  
- **Slack message not formatted**: use `mrkdwn` blocks, not plain `text` only for rich content.  
- **Teams card doesn’t render**: tenant may block Incoming Webhooks; use Power Automate or a bot.  
- **Discord file too large**: compress or host elsewhere and link.  
- **Email goes to spam**: use authenticated SMTP or a transactional provider, set SPF/DKIM/DMARC.

---

## 12) Recap
```plaintext
Load .env → choose channel → build payload (blocks/cards/embeds) → send with retries + backoff → dedupe noisy events → attach context/logs → monitor rate limits and delivery
```

**Next**: Use with monitors in `025_MONITORING_SCRIPTS.md`, schedule via `026_TASK_SCHEDULING.md`, and include summaries/graphs from `011_GRAPHS.md` or CSV exports from `013_CSVEXCEL.md`. 