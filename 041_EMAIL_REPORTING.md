

# Email Reporting: Generate and Send Automated Reports — updated Oct 6, 2025

Create automated **system** or **data** reports from CSV/Excel, render **HTML templates**, embed tables and charts, and **email** them on a schedule. This guide covers extracting metrics with **pandas**, rendering with **Jinja2**, attaching CSV/Excel artifacts, and sending via **SMTP** or **Microsoft Graph**. Pair with `013_CSVEXCEL.md` (data I/O), `011_GRAPHS.md` (visuals), and `014_EMAILS.md` (providers and auth).

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Data lives in `data/` as CSV or Excel (`.xlsx`).  
- Secrets via `.env` or Keychain.  
- You can use Gmail SMTP for personal tests or Graph for enterprise.

**Install**
```sh
pip install pandas jinja2 python-dotenv matplotlib openpyxl # openpyxl enables Excel write
# optional send methods
pip install msal requests  # Microsoft Graph
```

---

## Table of Contents
- [0) Flow and safety](#0-flow-and-safety)
- [1) Project layout](#1-project-layout)
- [2) Load CSV/Excel and compute metrics](#2-load-csvexcel-and-compute-metrics)
- [3) HTML templating with Jinja2](#3-html-templating-with-jinja2)
- [4) Charts and inline images](#4-charts-and-inline-images)
- [5) Compose the email: HTML + attachments](#5-compose-the-email-html--attachments)
- [6) Send via SMTP (personal)](#6-send-via-smtp-personal)
- [7) Send via Microsoft Graph (enterprise)](#7-send-via-microsoft-graph-enterprise)
- [8) Scheduling and automation](#8-scheduling-and-automation)
- [9) Testing and deliverability](#9-testing-and-deliverability)
- [10) Troubleshooting](#10-troubleshooting)
- [11) Recap](#11-recap)

---

## 0) Flow and safety
**What**: Define the end‑to‑end pipeline for reproducible reporting.  
**Why**: Clear boundaries reduce breakage and leak risk.

```plaintext
Load data → compute metrics → render HTML with template → attach artifacts → send email → log/send summary → schedule
```
Safety:
1. Keep secrets out of code (see `033_KEYCHAIN_SECRETS.md`).  
2. Validate inputs and handle empty data.  
3. Inline small images only; attach large artifacts.

---

## 1) Project layout
**What**: Keep templates, data, and code organized.  
**Why**: Simplifies reuse and templating.
```
reporting/
├── .env
├── make_report.py
├── mail_smtp.py
├── mail_graph.py
├── templates/
│   ├── report.html.j2
│   └── _styles.css
├── data/
│   ├── sales.csv
│   └── system_metrics.xlsx
└── out/
    ├── report.html
    ├── chart.png
    └── attachments/
```

---

## 2) Load CSV/Excel and compute metrics
**What**: Read input files and derive the KPIs your email will show.
```python
# file: make_report.py (part 1)
from __future__ import annotations
from pathlib import Path
import pandas as pd

DATA = Path("data/sales.csv")

def compute_metrics() -> dict:
    df = pd.read_csv(DATA)
    if df.empty:
        return {"empty": True}
    df["date"] = pd.to_datetime(df["date"])  # assumes a date column
    total = float(df["amount"].sum())
    by_prod = (
        df.groupby("product", as_index=False)["amount"].sum().sort_values("amount", ascending=False)
    )
    last7 = df[df["date"] >= df["date"].max() - pd.Timedelta(days=7)]["amount"].sum()
    return {"empty": False, "total": total, "by_product": by_prod, "last7": float(last7)}
```
Export a copy as Excel for recipients to explore:
```python
# file: make_report.py (part 2)
import pandas as pd
from pathlib import Path

def export_artifacts(by_prod: pd.DataFrame, outdir: Path) -> Path:
    outdir.mkdir(parents=True, exist_ok=True)
    xlsx = outdir/"breakdown.xlsx"
    with pd.ExcelWriter(xlsx, engine="openpyxl") as w:
        by_prod.to_excel(w, index=False, sheet_name="ByProduct")
    return xlsx
```

---

## 3) HTML templating with Jinja2
**What**: Render a styled HTML email using a template with placeholders.  
**Why**: Separates presentation from logic.
```python
# file: make_report.py (part 3)
from jinja2 import Environment, FileSystemLoader, select_autoescape

def render_html(context: dict, templates_dir: Path, out_html: Path) -> Path:
    env = Environment(
        loader=FileSystemLoader(str(templates_dir)),
        autoescape=select_autoescape(["html", "xml"]),
        trim_blocks=True, lstrip_blocks=True,
    )
    tpl = env.get_template("report.html.j2")
    html = tpl.render(**context)
    out_html.write_text(html, encoding="utf-8")
    return out_html
```
Template:
```html
<!-- file: templates/report.html.j2 -->
<!doctype html>
<html>
<head>
  <meta charset="utf-8"/>
  <style>
    body{font-family: -apple-system,Segoe UI,Roboto,Helvetica,Arial,sans-serif;}
    table{border-collapse:collapse;width:100%;}
    th,td{border:1px solid #ddd;padding:6px;text-align:left;font-size:13px}
    th{background:#f3f3f3}
    .kpi{font-size:16px;margin:6px 0}
  </style>
</head>
<body>
  <h2>{{ title }}</h2>
  <p class="kpi"><strong>Total:</strong> {{ total | round(2) }}</p>
  <p class="kpi"><strong>Last 7 days:</strong> {{ last7 | round(2) }}</p>
  {% if by_product is defined %}
  <h3>By product</h3>
  <table>
    <tr><th>Product</th><th>Amount</th></tr>
    {% for row in by_product.itertuples() %}
      <tr><td>{{ row.product }}</td><td>{{ '%.2f'|format(row.amount) }}</td></tr>
    {% endfor %}
  </table>
  {% endif %}
  {% if inline_img_cid %}
    <h3>Trend</h3>
    <img src="cid:{{ inline_img_cid }}" alt="chart"/>
  {% endif %}
</body>
</html>
```

---

## 4) Charts and inline images
**What**: Generate a small PNG and embed it inline with CID content‑id.  
**Why**: Quick visual without external hosting.
```python
# file: make_report.py (part 4)
import matplotlib.pyplot as plt

def make_chart(df: pd.DataFrame, out_png: Path) -> Path:
    # simple bar chart of top N products
    top = df.head(10)
    plt.figure()
    plt.bar(top["product"], top["amount"])  # default colors per style guide
    plt.xticks(rotation=30, ha='right')
    plt.tight_layout()
    plt.savefig(out_png, dpi=150)
    plt.close()
    return out_png
```

---

## 5) Compose the email: HTML + attachments
**What**: Build a MIME message with HTML body, inline image, and attachments.  
**Why**: Compatible with major clients.
```python
# file: make_report.py (part 5)
from email.message import EmailMessage
from email.utils import make_msgid
import mimetypes

def build_email(subject: str, html_path: Path, attachments: list[Path], inline_png: Path | None, sender: str, to: list[str]) -> EmailMessage:
    msg = EmailMessage()
    msg["Subject"] = subject
    msg["From"] = sender
    msg["To"] = ", ".join(to)

    html = html_path.read_text(encoding="utf-8")
    cid = None
    if inline_png and inline_png.exists():
        cid = make_msgid(domain="report.local")[1:-1]  # strip <>
    html = html.replace("{{ inline_img_cid }}", cid or "")
    msg.add_alternative(html, subtype="html")

    if inline_png and inline_png.exists() and cid:
        with open(inline_png, 'rb') as f:
            msg.get_payload()[0].add_related(f.read(), maintype='image', subtype='png', cid=f"<{cid}>")

    for p in attachments:
        if not p.exists():
            continue
        ctype, encoding = mimetypes.guess_type(p.name)
        maintype, subtype = (ctype or 'application/octet-stream').split('/', 1)
        msg.add_attachment(p.read_bytes(), maintype=maintype, subtype=subtype, filename=p.name)
    return msg
```

---

## 6) Send via SMTP (personal)
**What**: Use SMTP with STARTTLS.  
**Why**: Simple for personal Gmail or other providers.
```python
# file: mail_smtp.py
from __future__ import annotations
from dotenv import load_dotenv; load_dotenv()
import os, smtplib
from email.message import EmailMessage

def send(msg: EmailMessage) -> None:
    host = os.getenv("SMTP_HOST")
    user = os.getenv("SMTP_USER")
    pwd  = os.getenv("SMTP_PASS")
    port = int(os.getenv("SMTP_PORT", "587"))
    with smtplib.SMTP(host, port) as s:
        s.starttls(); s.login(user, pwd); s.send_message(msg)
```
`.env` example:
```
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=me@gmail.com
SMTP_PASS=app-password-here
```

---

## 7) Send via Microsoft Graph (enterprise)
**What**: Send as the signed‑in user or an app using Graph.  
**Why**: Better deliverability and compliance in Microsoft 365.
```python
# file: mail_graph.py
from __future__ import annotations
from dotenv import load_dotenv; load_dotenv()
import os, msal, requests

TENANT = os.getenv("AZURE_TENANT_ID", "common")
CLIENT_ID = os.getenv("AZURE_CLIENT_ID")
CLIENT_SECRET = os.getenv("AZURE_CLIENT_SECRET")  # for app-only
SCOPE = ["https://graph.microsoft.com/.default"]

# client credentials flow (app-only; requires Mail.Send permissions on the app)
app = msal.ConfidentialClientApplication(CLIENT_ID, authority=f"https://login.microsoftonline.com/{TENANT}", client_credential=CLIENT_SECRET)

def send_html(sender: str, to: list[str], subject: str, html: str):
    token = app.acquire_token_for_client(scopes=SCOPE)
    access = token["access_token"]
    body = {
      "message": {
        "subject": subject,
        "body": {"contentType": "HTML", "content": html},
        "toRecipients": [{"emailAddress": {"address": x}} for x in to],
        "from": {"emailAddress": {"address": sender}},
      },
      "saveToSentItems": True
    }
    r = requests.post("https://graph.microsoft.com/v1.0/users/"+sender+"/sendMail", headers={"Authorization": f"Bearer {access}"}, json=body, timeout=30)
    r.raise_for_status()
```
Notes:
- Attachments with Graph: add `attachments` array to the `message`. For large files, use Graph **attachment upload sessions**.
- For delegated flow (user sign‑in), use `PublicClientApplication` device code or auth code—see `014_EMAILS.md`.

---

## 8) Scheduling and automation
**What**: Run daily or weekly and email the results.  
**Why**: Hands‑off reports.

Driver script:
```python
# file: make_report.py (final)
from pathlib import Path
from dotenv import load_dotenv; load_dotenv()
import os

from make_report import compute_metrics, export_artifacts, render_html, make_chart, build_email  # same file sections

OUT = Path("out"); OUT.mkdir(exist_ok=True)
ctx = compute_metrics()
if ctx.get("empty"):
    raise SystemExit("no data")
chart = make_chart(ctx["by_product"], OUT/"chart.png")
ctx_full = {**ctx, "title": os.getenv("REPORT_TITLE", "Weekly Sales Report"), "inline_img_cid": "INLINE_CID_PLACEHOLDER"}
html = render_html(ctx_full, Path("templates"), OUT/"report.html")
attachment = export_artifacts(ctx["by_product"], OUT/"attachments")

msg = build_email(
    subject=os.getenv("REPORT_SUBJECT", "Weekly Sales"),
    html_path=html,
    attachments=[attachment],
    inline_png=chart,
    sender=os.getenv("MAIL_FROM"),
    to=os.getenv("MAIL_TO", "me@example.com").split(",")
)

if os.getenv("USE_GRAPH") == "1":
    from mail_graph import send_html
    send_html(os.getenv("MAIL_FROM"), os.getenv("MAIL_TO").split(","), msg["Subject"], html.read_text("utf-8"))
else:
    from mail_smtp import send
    send(msg)
```
Schedule with `026_TASK_SCHEDULING.md` (launchd, cron, or APScheduler). Log a summary and notify failures with `031_NOTIFICATION_AUTOMATION.md`.

---

## 9) Testing and deliverability
**What**: Validate rendering and reduce spam risk.  
**Why**: Better open rates and fewer bounces.
- Open `out/report.html` in a browser before sending.  
- Keep HTML simple; avoid external CSS and JS.  
- Prefer **inline styles** and tables for layout; most clients are limited.  
- For Gmail/Outlook, avoid base64 mega‑images in the body.  
- Ensure SPF/DKIM/DMARC where applicable (enterprise IT handles this).

---

## 10) Troubleshooting
- **Blank email / images missing**: Inline image CID not referenced or not added as `related`. Replace the template placeholder and ensure `add_related` is used.  
- **Attachments corrupted**: Open files in binary mode and pass correct MIME type.  
- **SMTP 535 / auth failed**: Use an app password or OAuth; confirm host/port.  
- **Graph 401/403**: App missing `Mail.Send` permission or wrong tenant; consent required.  
- **Excel open warning**: Ensure `.xlsx` created with `openpyxl` and sheets named properly.  
- **Large emails rejected**: Trim inline images, compress attachments, or move artifacts to a link.

---

## 11) Recap
```plaintext
Read data (CSV/Excel) → compute metrics → render HTML with Jinja2 → add inline chart + attach artifacts → send via SMTP or Graph → schedule and monitor
```

**Next**: Store credentials in Keychain (`033_KEYCHAIN_SECRETS.md`), visualize trends in `011_GRAPHS.md`, and reuse your packaging patterns from `035_SCRIPT_PACKAGING.md` to ship `reporting` as a CLI.