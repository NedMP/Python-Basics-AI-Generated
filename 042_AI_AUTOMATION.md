

# AI Automation with OpenAI & Azure OpenAI — updated Oct 6, 2025

Use large language models (LLMs) to summarize, parse, and analyze text; automate documentation; and boost support workflows. This note shows **OpenAI** and **Azure OpenAI** patterns using Python: auth, prompting, structured JSON output, batching, streaming, embeddings for search, and safety. Pair with `002_SETUP.md` for venv and `.env` management.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Secrets in `.env` or macOS Keychain (`033_KEYCHAIN_SECRETS.md`).  
- You have an OpenAI API key **or** an Azure OpenAI resource + deployment.

**Install**
```sh
pip install requests python-dotenv tiktoken tenacity
# optional: for embeddings + local vector search
pip install numpy faiss-cpu
```

---

## Table of Contents
- [0) OpenAI vs Azure OpenAI](#0-openai-vs-azure-openai)
- [1) Environment and configuration](#1-environment-and-configuration)
- [2) Quick start: summarize text](#2-quick-start-summarize-text)
- [3) Structured JSON parsing](#3-structured-json-parsing)
- [4) Streaming tokens](#4-streaming-tokens)
- [5) Batch processing and retries](#5-batch-processing-and-retries)
- [6) Embeddings + local search (RAG-lite)](#6-embeddings--local-search-rag-lite)
- [7) Automating documentation from Markdown](#7-automating-documentation-from-markdown)
- [8) Support workflows: classify, triage, draft replies](#8-support-workflows-classify-triage-draft-replies)
- [9) Azure OpenAI specifics](#9-azure-openai-specifics)
- [10) Token limits, cost, and safety](#10-token-limits-cost-and-safety)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) OpenAI vs Azure OpenAI
**What**: Both expose GPT‑class models via HTTP.  
**Why**: Choose based on compliance, tenancy, and network controls.
- **OpenAI**: fastest access to new models; single global endpoint.  
- **Azure OpenAI**: enterprise controls, VNET, private endpoints; you deploy a model as a **deployment name** and call it via your resource endpoint.

---

## 1) Environment and configuration
**What**: Centralize secrets and model names.  
**Why**: Swap providers without code edits.

`.env` examples:
```
# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini

# Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://my-resource.openai.azure.com
AZURE_OPENAI_API_KEY=...
AZURE_OPENAI_DEPLOYMENT=gpt-4o-mini  # deployment name you created
AZURE_OPENAI_API_VERSION=2024-08-01-preview
```
Loader:
```python
from dotenv import load_dotenv; load_dotenv()
import os
CFG = {
  "openai": {
    "key": os.getenv("OPENAI_API_KEY"),
    "model": os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
  },
  "azure": {
    "endpoint": os.getenv("AZURE_OPENAI_ENDPOINT"),
    "key": os.getenv("AZURE_OPENAI_API_KEY"),
    "deployment": os.getenv("AZURE_OPENAI_DEPLOYMENT"),
    "api_version": os.getenv("AZURE_OPENAI_API_VERSION", "2024-08-01-preview"),
  }
}
```

---

## 2) Quick start: summarize text
**What**: Send a prompt and receive a concise summary.  
**Why**: Baseline call pattern.
```python
# file: ai_summary.py
from __future__ import annotations
from dotenv import load_dotenv; load_dotenv()
import os, requests

TEXT = """Your long text goes here..."""

# OpenAI Chat Completions
r = requests.post(
  "https://api.openai.com/v1/chat/completions",
  headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"},
  json={
    "model": os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
    "messages": [
      {"role":"system","content":"You are a terse technical summarizer."},
      {"role":"user","content": f"Summarize in 5 bullets:\n{TEXT}"}
    ]
  }, timeout=60)
r.raise_for_status()
print(r.json()["choices"][0]["message"]["content"])
```

---

## 3) Structured JSON parsing
**What**: Force machine‑readable output for pipelines.  
**Why**: Avoid fragile string parsing.
```python
# file: ai_structured.py
from __future__ import annotations
from dotenv import load_dotenv; load_dotenv()
import os, requests, json

schema = {
  "type": "object",
  "properties": {
    "title": {"type":"string"},
    "priority": {"type":"string", "enum":["low","medium","high"]},
    "tags": {"type":"array", "items": {"type":"string"}}
  },
  "required": ["title","priority"]
}

prompt = """
Extract a concise title, a priority, and a few tags from the incident text.
Return ONLY JSON that validates against the schema.
Incident: VPN outage for APAC users; degraded since 09:00; spike in 5xx.
"""

r = requests.post(
  "https://api.openai.com/v1/chat/completions",
  headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"},
  json={
    "model": os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
    "response_format": {"type":"json_object"},
    "messages":[{"role":"user","content": prompt}]
  }, timeout=60)
obj = json.loads(r.json()["choices"][0]["message"]["content"])
print(obj)
```
Tip: Validate with `jsonschema` if strictness is required.

---

## 4) Streaming tokens
**What**: Receive partial output progressively.  
**Why**: Better UX and early feedback for long generations.
```python
import os, requests
url = "https://api.openai.com/v1/chat/completions"
headers = {"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"}
json = {
  "model": os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
  "stream": True,
  "messages": [
    {"role":"system","content":"Stream concise responses."},
    {"role":"user","content":"List three remediation steps for high disk usage."}
  ]
}
with requests.post(url, headers=headers, json=json, stream=True, timeout=300) as r:
    r.raise_for_status()
    for line in r.iter_lines(decode_unicode=True):
        if not line or not line.startswith("data:"): continue
        if line.strip() == "data: [DONE]": break
        chunk = line.removeprefix("data: ")
        print(chunk)
```
The stream yields JSON chunks with deltas; buffer then render.

---

## 5) Batch processing and retries
**What**: Process many items safely.  
**Why**: Rate limits and transient errors are normal.
```python
# file: ai_batch.py
from __future__ import annotations
from tenacity import retry, stop_after_attempt, wait_exponential_jitter
import os, requests

@retry(stop=stop_after_attempt(6), wait=wait_exponential_jitter(initial=0.5, max=20))
def call(messages):
    r = requests.post(
      "https://api.openai.com/v1/chat/completions",
      headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"},
      json={"model": os.getenv("OPENAI_MODEL", "gpt-4o-mini"), "messages": messages}, timeout=60)
    if r.status_code in (429, 500, 502, 503, 504):
        raise RuntimeError(f"retryable: {r.status_code}")
    r.raise_for_status(); return r.json()["choices"][0]["message"]["content"]

texts = ["doc1 text...", "doc2 text..."]
outputs = [call([{ "role":"user", "content": f"Summarize: {t}" }]) for t in texts]
print(outputs)
```
Batch smaller inputs client‑side to stay within token limits.

---

## 6) Embeddings + local search (RAG-lite)
**What**: Convert text to vectors and search locally before calling the LLM.  
**Why**: Cheaper and more accurate answers against your own notes.
```python
# file: rag_lite.py
from __future__ import annotations
from dotenv import load_dotenv; load_dotenv()
import os, requests, faiss, numpy as np

EMBED_MODEL = os.getenv("OPENAI_EMBED_MODEL", "text-embedding-3-small")

# 1) create embeddings
texts = [
  ("002_SETUP.md", open("002_SETUP.md","r",encoding="utf-8").read()),
  ("000_STRUCTURE.md", open("000_STRUCTURE.md","r",encoding="utf-8").read()),
]

payload = {"model": EMBED_MODEL, "input": [t for _, t in texts]}
r = requests.post("https://api.openai.com/v1/embeddings",
                  headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"},
                  json=payload, timeout=120)
r.raise_for_status()
vecs = np.array([item["embedding"] for item in r.json()["data"]], dtype="float32")

# 2) index with FAISS and search
index = faiss.IndexFlatIP(vecs.shape[1])
# normalize for cosine similarity
faiss.normalize_L2(vecs)
index.add(vecs)

query = "How do I set up pyenv on macOS?"
qr = requests.post("https://api.openai.com/v1/embeddings",
                   headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"},
                   json={"model": EMBED_MODEL, "input": query}).json()
qv = np.array(qr["data"][0]["embedding"], dtype="float32"); faiss.normalize_L2(qv.reshape(1,-1))
D, I = index.search(qv.reshape(1,-1), k=2)
context = "\n\n".join(texts[i][1][:2000] for i in I[0])

# 3) ask with context
ans = requests.post("https://api.openai.com/v1/chat/completions",
  headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"},
  json={
    "model": os.getenv("OPENAI_MODEL", "gpt-4o-mini"),
    "messages": [
      {"role":"system","content":"Answer using only the provided context. If unknown, say you don't know."},
      {"role":"user","content": f"Context:\n{context}\n\nQuestion: {query}"}
    ]
  }).json()["choices"][0]["message"]["content"]
print(ans)
```

---

## 7) Automating documentation from Markdown
**What**: Turn raw notes into summaries, outlines, or change logs.  
**Why**: Speeds up maintenance of docs like this repo.
```python
# file: doc_automation.py
from pathlib import Path
import requests, os

md = Path("002_SETUP.md").read_text(encoding="utf-8")
prompt = f"""
Summarize the following Markdown into 5 bullets for a README changelog. Keep file references and section numbers.
---
{md[:12000]}
"""

r = requests.post("https://api.openai.com/v1/chat/completions",
  headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"},
  json={"model": os.getenv("OPENAI_MODEL", "gpt-4o-mini"), "messages":[{"role":"user","content": prompt}]}, timeout=120)
print(r.json()["choices"][0]["message"]["content"])
```
Integrate with `037_GIT_AUTOMATION.md` to auto‑commit summaries.

---

## 8) Support workflows: classify, triage, draft replies
**What**: Extract fields from tickets and suggest responses.  
**Why**: Faster first response and consistent tagging.
```python
TICKET = "User cannot log into VPN from macOS 14. Error: authentication failed."  # example

prompt = """
Classify the ticket with fields {category, severity, product}. Then draft a 3‑paragraph reply with steps and one follow‑up question.
Return JSON with {category, severity, product, reply}.
"""

res = requests.post("https://api.openai.com/v1/chat/completions",
  headers={"Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}", "Content-Type":"application/json"},
  json={"model": os.getenv("OPENAI_MODEL", "gpt-4o-mini"), "response_format": {"type":"json_object"},
        "messages":[{"role":"user","content": prompt + "\n\nTicket: " + TICKET}]}
).json()
print(res["choices"][0]["message"]["content"])
```
Wire this into Slack/Teams via `031_NOTIFICATION_AUTOMATION.md`.

---

## 9) Azure OpenAI specifics
**What**: Same patterns, different base URL and **deployment**.
```python
# file: azure_chat.py
from __future__ import annotations
from dotenv import load_dotenv; load_dotenv()
import os, requests

base = os.getenv("AZURE_OPENAI_ENDPOINT")
ver = os.getenv("AZURE_OPENAI_API_VERSION", "2024-08-01-preview")
deploy = os.getenv("AZURE_OPENAI_DEPLOYMENT")

url = f"{base}/openai/deployments/{deploy}/chat/completions?api-version={ver}"
headers = {"api-key": os.getenv("AZURE_OPENAI_API_KEY"), "Content-Type": "application/json"}

r = requests.post(url, headers=headers, json={
  "messages": [{"role":"user","content":"Give me a one‑line summary of pyenv."}],
  "temperature": 0.2
}, timeout=60)
print(r.json()["choices"][0]["message"]["content"])
```
Embedding endpoint is similar: `/embeddings?api-version=...` with a deployment of an embedding model.

---

## 10) Token limits, cost, and safety
**What**: Control risk and spend.  
**Why**: Predictable, compliant automation.
- **Token budgeting**: truncate or chunk inputs; keep prompts minimal. Use `tiktoken` to estimate tokens.  
- **PII**: do not paste secrets; mask emails, IPs, and IDs when possible.  
- **Determinism**: use low `temperature` for predictable outputs.  
- **Guardrails**: demand JSON via `response_format` and validate.  
- **Rate limits**: exponential backoff; respect `Retry-After`.  
- **Caching**: hash inputs and reuse outputs to avoid re‑billing.  
- **Audit**: log prompts and outputs, but **never** raw secrets (see `016_LOGGING.md`).

Token estimate example:
```python
import tiktoken
enc = tiktoken.get_encoding("cl100k_base")
print(len(enc.encode("How many tokens am I?")))
```

---

## 11) Troubleshooting
- **401/403**: bad key, wrong endpoint, or disabled deployment.  
- **404 on Azure**: model not deployed or wrong deployment name.  
- **429**: throttled. Backoff and lower concurrency.  
- **`Invalid URL`**: double check base URL and `api-version` on Azure.  
- **JSON parse errors**: enforce `response_format` and validate.  
- **Timeouts**: increase `timeout` and reduce input size.

---

## 12) Recap
```plaintext
Pick provider → load .env → send minimal prompts → prefer JSON outputs → stream when useful → batch with retries → add embeddings for search → watch tokens, PII, and rate limits
```

**Next**: Pipe results into `041_EMAIL_REPORTING.md` for reporting, trigger notifications via `031_NOTIFICATION_AUTOMATION.md`, and schedule nightly documentation runs with `026_TASK_SCHEDULING.md`. 