

# Sending and Automating Emails with Python — updated Oct 5, 2025

Beginner‑friendly guide for automating email from Python. Two paths:
- **Personal**: Gmail (Gmail API preferred; SMTP with App Password as fallback).
- **Enterprise**: Microsoft 365 (Exchange Online) via **Microsoft Graph** with an Azure App.

Use with `002_SETUP.md` for environment setup and `.env` handling.

---

**Assumptions and Conventions**
- macOS + zsh, Python 3.12+ or 3.13, virtual environment active.  
- Secrets come from `.env` using `python-dotenv`.  
- All examples are copy‑pasteable starter snippets. Adjust scopes, tenants, and policies per your org.

---

## Table of Contents
- [0) Choose the right approach](#0-choose-the-right-approach)
- [1) Common project scaffold](#1-common-project-scaffold)
- [2) Personal: Gmail via Gmail API (OAuth)](#2-personal-gmail-via-gmail-api-oauth)
- [3) Personal: Gmail via SMTP (App Password)](#3-personal-gmail-via-smtp-app-password)
- [4) Enterprise: Microsoft 365 via Graph — delegated user (Device Code)](#4-enterprise-microsoft-365-via-graph--delegated-user-device-code)
- [5) Enterprise: Microsoft 365 via Graph — daemon/service (Client Credentials)](#5-enterprise-microsoft-365-via-graph--daemonservice-client-credentials)
- [6) Attachments and HTML](#6-attachments-and-html)
- [7) Scheduling and retries](#7-scheduling-and-retries)
- [8) Deliverability and safety](#8-deliverability-and-safety)
- [9) Troubleshooting](#9-troubleshooting)
- [10) Recap](#10-recap)

---

## 0) Choose the right approach
**Gmail (personal projects):**
- Prefer the **Gmail API**. OAuth grants scoped access and avoids password handling.  
- SMTP works with **App Passwords** only if 2‑Step Verification is enabled. Some accounts disable this.

**Microsoft 365 / Exchange Online (work):**
- Use **Microsoft Graph**. For user‑present apps, use **delegated** auth (device code or interactive). For automation/daemons, use **client credentials** with **application** permissions.

---

## 1) Common project scaffold
Create a minimal structure and install deps.
```sh
mkdir -p ~/Code_Stuff/email-demo && cd ~/Code_Stuff/email-demo
python -m venv .venv && source .venv/bin/activate
pip install python-dotenv rich

# Personal Gmail API
pip install google-auth google-auth-oauthlib google-api-python-client

# Enterprise Graph
pip install msal requests

# Optional extras
pip install jinja2 tenacity
```
`.env` template:
```
# Gmail API
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=
GOOGLE_PROJECT_ID=

# SMTP fallback
GMAIL_ADDRESS=
GMAIL_APP_PASSWORD=

# Microsoft Graph (delegated)
AZURE_CLIENT_ID=
AZURE_TENANT_ID=

# Microsoft Graph (daemon)
AZURE_CC_CLIENT_ID=
AZURE_CC_TENANT_ID=
AZURE_CC_CLIENT_SECRET=
# or path to certificate if using cert-based auth
```

---

## 2) Personal: Gmail via Gmail API (OAuth)
**What**: Use Google’s official API to send mail as the signed‑in user.  
**Why**: Modern, secure, scoped access; avoids storing passwords.

### 2.1 Create OAuth client and enable API
1. Go to **Google Cloud Console** → create/select a project.  
2. **Enable APIs & Services** → **Gmail API**.  
3. **Credentials** → **Create Credentials** → **OAuth client ID** → Application type: **Desktop**.  
4. Download `client_secret_*.json` and keep it private.

### 2.2 First send with the Gmail API
`gmail_send.py`:
```python
from __future__ import annotations
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from base64 import urlsafe_b64encode
from pathlib import Path
from dotenv import load_dotenv
import os

from google.oauth2.credentials import Credentials
from google_auth_oauthlib.flow import InstalledAppFlow
from googleapiclient.discovery import build

# Scopes: readonly → send
SCOPES = ["https://www.googleapis.com/auth/gmail.send"]

load_dotenv()
CLIENT_SECRETS = Path("client_secret.json")  # rename your downloaded file
TOKEN_FILE = Path("token.json")              # stores refresh/access tokens

def get_service():
    creds = None
    if TOKEN_FILE.exists():
        creds = Credentials.from_authorized_user_file(TOKEN_FILE, SCOPES)
    if not creds or not creds.valid:
        if creds and creds.expired and creds.refresh_token:
            creds.refresh(Request())  # type: ignore[name-defined]
        else:
            flow = InstalledAppFlow.from_client_secrets_file(str(CLIENT_SECRETS), SCOPES)
            creds = flow.run_local_server(port=0)
        TOKEN_FILE.write_text(creds.to_json())
    return build("gmail", "v1", credentials=creds)

def as_raw_message(subject: str, sender: str, to: str, html: str | None = None, text: str | None = None):
    if html:
        msg = MIMEMultipart("alternative")
        if text:
            msg.attach(MIMEText(text, "plain", "utf-8"))
        msg.attach(MIMEText(html, "html", "utf-8"))
    else:
        msg = MIMEText(text or "", "plain", "utf-8")
    msg["To"], msg["From"], msg["Subject"] = to, sender, subject
    raw = urlsafe_b64encode(msg.as_bytes()).decode("ascii")
    return {"raw": raw}

if __name__ == "__main__":
    service = get_service()
    body = as_raw_message(
        subject="Hello from Python",
        sender=os.getenv("GMAIL_ADDRESS", "me"),
        to=os.getenv("GMAIL_ADDRESS", "me"),
        text="This is a test sent via Gmail API.",
        html="<p><strong>This</strong> is a test via Gmail API.</p>")
    res = service.users().messages().send(userId="me", body=body).execute()
    print("Sent id:", res.get("id"))
```
Run once to authorize in the browser; tokens persist in `token.json` for silent reuse.

> Tip: To read inbox, add scope `gmail.readonly` and use `users().messages().list()`.

---

## 3) Personal: Gmail via SMTP (App Password)
**What**: Send via `smtp.gmail.com` using an **App Password**.  
**When**: Quick scripts where the Gmail API is overkill. Requires 2‑Step Verification and an app password.

`gmail_smtp.py`:
```python
import smtplib, ssl
from email.message import EmailMessage
from dotenv import load_dotenv
import os

load_dotenv()
addr = os.getenv("GMAIL_ADDRESS")
app_pw = os.getenv("GMAIL_APP_PASSWORD")

msg = EmailMessage()
msg["From"], msg["To"], msg["Subject"] = addr, addr, "SMTP test"
msg.set_content("Plain text body")
msg.add_alternative("""<h3>HTML</h3><p>SMTP via app password.</p>""", subtype="html")

context = ssl.create_default_context()
with smtplib.SMTP_SSL("smtp.gmail.com", 465, context=context) as s:
    s.login(addr, app_pw)
    s.send_message(msg)
print("sent")
```
If login fails, confirm 2‑Step Verification is on and the **App Password** exists. Regular passwords won’t work.

---

## 4) Enterprise: Microsoft 365 via Graph — delegated user (Device Code)
**What**: User‑present script. Authenticates a user and sends as that user.  
**Why**: Ideal for tools run by humans without storing passwords.

### 4.1 Azure App registration (once)
1. Azure Portal → **Microsoft Entra ID** → **App registrations** → **New registration**.  
2. Note **Application (client) ID** and **Directory (tenant) ID**.  
3. **API permissions** → **Microsoft Graph** → **Delegated permissions** → add `Mail.Send`.  
4. **Grant admin consent** if your tenant requires it.

### 4.2 Device code + sendMail
`graph_send_delegated.py`:
```python
import os, requests
from dotenv import load_dotenv
import msal

load_dotenv()
CLIENT_ID = os.getenv("AZURE_CLIENT_ID")
TENANT_ID = os.getenv("AZURE_TENANT_ID")
AUTHORITY = f"https://login.microsoftonline.com/{TENANT_ID}"
SCOPES = ["https://graph.microsoft.com/.default"]  # for app-configured perms

app = msal.PublicClientApplication(client_id=CLIENT_ID, authority=AUTHORITY)
flow = app.initiate_device_flow(scopes=["Mail.Send"])
if "user_code" not in flow:
    raise RuntimeError("Device flow failed to start")
print("Open:", flow["verification_uri"])
print("Code:", flow["user_code"])
result = app.acquire_token_by_device_flow(flow)
if "access_token" not in result:
    raise RuntimeError(result.get("error_description"))

access_token = result["access_token"]
headers = {"Authorization": f"Bearer {access_token}", "Content-Type": "application/json"}

payload = {
  "message": {
    "subject": "Graph delegated send",
    "body": {"contentType": "HTML", "content": "<p>Hello from <b>delegated</b> flow.</p>"},
    "toRecipients": [{"emailAddress": {"address": os.getenv("GMAIL_ADDRESS", "someone@example.com")}}]
  },
  "saveToSentItems": True
}

resp = requests.post("https://graph.microsoft.com/v1.0/me/sendMail", headers=headers, json=payload)
resp.raise_for_status()
print("sent")
```
Run and follow the printed URL/code to sign in.

---

## 5) Enterprise: Microsoft 365 via Graph — daemon/service (Client Credentials)
**What**: Headless automation. App sends mail without user interaction.  
**Why**: Nightly jobs, alerts, integrations. Requires **Application** permission `Mail.Send` and approval.

### 5.1 App registration and permissions
1. Register app as above.  
2. **Certificates & secrets** → create a **client secret** or upload a certificate.  
3. **API permissions** → **Microsoft Graph** → **Application permissions** → add `Mail.Send`.  
4. **Grant admin consent**.  
5. Choose a mailbox to send as. With application perms you typically send as a specific mailbox using `/users/{id-or-upn}/sendMail`.

### 5.2 Client credentials + send as shared mailbox
`graph_send_daemon.py`:
```python
import os, requests
from dotenv import load_dotenv
import msal

load_dotenv()
TENANT = os.getenv("AZURE_CC_TENANT_ID")
CLIENT = os.getenv("AZURE_CC_CLIENT_ID")
SECRET = os.getenv("AZURE_CC_CLIENT_SECRET")
AUTHORITY = f"https://login.microsoftonline.com/{TENANT}"
SCOPE = ["https://graph.microsoft.com/.default"]

app = msal.ConfidentialClientApplication(CLIENT, authority=AUTHORITY, client_credential=SECRET)
result = app.acquire_token_for_client(scopes=SCOPE)
if "access_token" not in result:
    raise RuntimeError(result.get("error_description"))

token = result["access_token"]
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

mailbox = "shared-mailbox@yourorg.com"  # or user UPN
payload = {
  "message": {
    "subject": "Graph daemon send",
    "body": {"contentType": "HTML", "content": "<p>Automated daemon message.</p>"},
    "toRecipients": [{"emailAddress": {"address": "someone@example.com"}}]
  },
  "saveToSentItems": True
}

url = f"https://graph.microsoft.com/v1.0/users/{mailbox}/sendMail"
resp = requests.post(url, headers=headers, json=payload)
resp.raise_for_status()
print("sent")
```
> Many tenants restrict application‑permission sending. Work with admins to scope mailboxes and apply Conditional Access as required.

---

## 6) Attachments and HTML
**Gmail API**: include attachments by building a `MIMEMultipart` with `MIMEBase` parts then base64url‑encoding the raw bytes.  
**Graph**: use the `attachments` array in the JSON payload or send MIME via `/sendMail` with a raw message.

Graph JSON attachment snippet:
```python
payload = {
  "message": {
    "subject": "Report",
    "body": {"contentType": "HTML", "content": "<p>See attachment.</p>"},
    "toRecipients": [{"emailAddress": {"address": "someone@example.com"}}],
    "attachments": [{
      "@odata.type": "#microsoft.graph.fileAttachment",
      "name": "report.csv",
      "contentType": "text/csv",
      "contentBytes": "UEssQ0xJ...base64..."  # base64 of file bytes
    }]
  },
  "saveToSentItems": True
}
```
For dynamic bodies, render HTML via **Jinja2** templates and inline CSS if needed.

---

## 7) Scheduling and retries
- Use **cron** (macOS launchd) or a CI runner for periodic sends.  
- Add **retries with backoff** on 429/5xx using `tenacity` or `urllib3 Retry`.  
- Log message IDs and request IDs for audits.  
- Keep tokens cached (`token.json` for Gmail, MSAL token cache for Graph).

---

## 8) Deliverability and safety
- Prefer verified sender domains (SPF, DKIM, DMARC).  
- Avoid spammy content and sudden volume spikes. Warm up gradually.  
- Never hardcode secrets. Load from `.env` or a vault.  
- Respect org policies. Some tenants block external recipients by default.  
- For bulk mail, use dedicated providers (SES, SendGrid) to protect your main domain reputation.

---

## 9) Troubleshooting
- **Gmail `invalid_grant`**: token revoked or expired → delete `token.json` and re‑authorize.  
- **Gmail `535-5.7.8` with SMTP**: App Password missing; standard passwords are blocked.  
- **Graph `401/403`**: permissions not consented or wrong flow (delegated vs application).  
- **Graph throttling 429**: implement retry‑after.  
- **HTML renders oddly**: email clients vary; test with plain text fallback and simple, inline CSS.  
- **Attachments corrupted**: base64 encoding missing or wrong content type.

---

## 10) Recap
```plaintext
Personal → Gmail API (OAuth) or SMTP with App Password
Enterprise → Microsoft Graph (delegated for user‑present, client credentials for daemons)
Add HTML/attachments → test → schedule + retry → monitor deliverability and permissions
```

**Next**: Wrap these snippets in functions, add Jinja2 templates for bodies, build a CLI with `argparse` for batch sends, and secure secrets via 1Password/Vault or platform keychains.