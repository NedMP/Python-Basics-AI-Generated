

# Secure Secrets on macOS (Keychain, keyring, env vaults) — updated Oct 6, 2025

How to store and retrieve credentials **safely** on macOS. Focus on **macOS Keychain** (system-native), Python’s **`keyring`** API, and **environment vaults** (e.g., 1Password CLI, `dotenv`, `direnv`). Includes patterns for apps, CLIs, and scheduled scripts. Pair with `002_SETUP.md` for venv and `.env` hygiene.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- You avoid hardcoding secrets or committing them to Git.  
- For enterprise, prefer org-managed vaults (Keychain, 1Password, Azure Key Vault) over plain `.env`.

**Install**
```sh
pip install keyring python-dotenv
# optional helpers
brew install jq
brew install --cask 1password 1password-cli # if using 1Password
```

---

## Table of Contents
- [0) Principles and threat model](#0-principles-and-threat-model)
- [1) Option A: macOS Keychain via Python `keyring`](#1-option-a-macos-keychain-via-python-keyring)
- [2) Option B: macOS `security` CLI](#2-option-b-macos-security-cli)
- [3) Option C: Environment vaults (`.env`, `direnv`, 1Password CLI)](#3-option-c-environment-vaults-env-direnv-1password-cli)
- [4) Choosing identifiers, rotation, and scopes](#4-choosing-identifiers-rotation-and-scopes)
- [5) Using secrets in scripts and apps](#5-using-secrets-in-scripts-and-apps)
- [6) Scheduling and headless automation](#6-scheduling-and-headless-automation)
- [7) Auditing, backup, and migration](#7-auditing-backup-and-migration)
- [8) Troubleshooting](#8-troubleshooting)
- [9) Recap](#9-recap)

---

## 0) Principles and threat model
**What**: Treat secrets as volatile inputs, not code.  
**Why**: Minimizes leakage and supports rotation.
- Prefer **OS keychain** or **vault** over plaintext files.  
- Scope each secret to the minimum required app/account.  
- Rotate regularly and log access in your app.  
- Never print secrets; redact in logs.

---

## 1) Option A: macOS Keychain via Python `keyring`
**What**: Python API that talks to the system Keychain using the native backend.  
**Why**: Simple, secure, no extra daemons.

### 1.1 Store and read a password/token
```python
# file: kc_demo.py
import keyring
SERVICE = "py-notes-demo"   # logical app/service name
USER = "ned@example.com"    # account identifier

# set/overwrite
keyring.set_password(SERVICE, USER, "s3cr3t-token-abc")

# get
token = keyring.get_password(SERVICE, USER)
print("len:", len(token))
```

### 1.2 Delete and list
```python
import keyring
keyring.delete_password("py-notes-demo", "ned@example.com")
```
> Listing items is restricted by the backend; use Keychain Access app or the `security` CLI for inspection.

### 1.3 Backend check and fallback
```python
import keyring
print(type(keyring.get_keyring()).__name__)  # should be macOSKeychain
```
If it falls back to **PlaintextKeyring**, the Keychain is unavailable. Fix by installing a supported backend or running from a normal desktop session (not a restricted container).

---

## 2) Option B: macOS `security` CLI
**What**: Native tool for scripting Keychain.  
**Why**: Useful for bootstrapping, CI on a logged-in macOS agent, or migrating items.

### 2.1 Create/update an Internet password
```sh
# add or update (-U) an internet password item
security add-internet-password -a ned@example.com -s login.example.com -w 's3cr3t' -U
```

### 2.2 Find and print
```sh
security find-internet-password -a ned@example.com -s login.example.com -w
```

### 2.3 Generic password items
```sh
security add-generic-password -a ned -s py-notes-demo -w 'api-key-123' -U
security find-generic-password -s py-notes-demo -w
```

**Notes**
- First access may prompt for permission. Approve once and macOS will remember.  
- Use **Keychain Access** app to adjust *Access Control* (which apps can read).  
- For unattended use, ensure the user session is unlocked.

---

## 3) Option C: Environment vaults (`.env`, `direnv`, 1Password CLI)
**What**: Load secrets into **process environment** only at runtime.  
**Why**: Keeps code portable and avoids storing values in repo.

### 3.1 `.env` with `python-dotenv` (local dev)
```env
# .env
API_BASE=https://api.example.com
API_TOKEN=op://MyVault/API/api-token    # store reference or dummy
```
```python
from dotenv import load_dotenv; load_dotenv()
import os
api_token = os.getenv("API_TOKEN")
```
> Prefer storing the *real* secret in Keychain or 1Password and placing a **reference** in `.env`.

### 3.2 `direnv` to auto-load env vars
```sh
brew install direnv
# .envrc in project root
use dotenv
```
Allow once:
```sh
direnv allow .
```

### 3.3 1Password CLI (op) injection
```sh
# login once
op account add --address my.1password.com --email ned@example.com
op signin --raw > ~/.op_session

# export a secret at runtime (no plaintext on disk)
export API_TOKEN=$(op read "op://MyVault/API/api-token")
python app.py
```
**Tip**: Avoid writing secrets to files; export just-in-time, then unset.

---

## 4) Choosing identifiers, rotation, and scopes
**What**: Consistent naming and lifecycles.  
**Why**: Easier audits and safer automation.
- **Service name**: app or domain, e.g., `py-notes-demo` or `corp-billing-api`.  
- **Account**: email, username, or env (`prod`, `dev`).  
- **Rotation**: maintain `set_password` wrappers that log rotate events (without values).  
- **Scopes**: token per environment; least privilege for each.

---

## 5) Using secrets in scripts and apps
**What**: Fetch once, keep in memory, and pass to clients.  
**Why**: Limit exposure and accidental logging.
```python
# file: use_secret.py
import keyring, os
from dotenv import load_dotenv; load_dotenv()

svc = os.getenv("SERVICE_NAME", "corp-api")
acct = os.getenv("SERVICE_ACCOUNT", "ned@example.com")
api_key = keyring.get_password(svc, acct)
if not api_key:
    raise SystemExit("No secret in Keychain; set with keyring.set_password")

import requests
r = requests.get("https://api.example.com/health", headers={"Authorization": f"Bearer {api_key}"}, timeout=10)
print(r.status_code)
```

**Logging**: log only hashes or last 4 chars; never the full secret. Use `016_LOGGING.md` patterns.

---

## 6) Scheduling and headless automation
**What**: run from cron/launchd without prompts.
- Use **per-user LaunchAgents** (see `026_TASK_SCHEDULING.md`). They run within the logged-in session so Keychain is unlocked.  
- For cron in non-GUI sessions, Keychain may be locked. Prefer launchd or store secrets in a vault with CLI retrieval at runtime (e.g., 1Password `op read`).  
- For CI on mac build agents, pre-create items and ensure the agent user has an unlocked login keychain.

---

## 7) Auditing, backup, and migration
**What**: Keep track of where secrets live and recover in disasters.
- Document **service/account** pairs and rotation owners.  
- Use `security dump-keychain` for inventories (do not commit outputs).  
- Keychain is included in Time Machine if the login keychain is unlocked during backup; still keep vault-of-record elsewhere.  
- For cross-platform teams, prefer a shared vault (1Password, Azure Key Vault) and load into macOS at runtime.

---

## 8) Troubleshooting
- **`PlaintextKeyring` in use**: environment lacks Keychain access; run in a user session or install a supported backend.  
- **Access prompt loops**: open *Keychain Access* → item → *Access Control* → allow your Python binary or Terminal.  
- **`security: SecKeychain...` errors**: keychain locked; unlock via Keychain Access or `security unlock-keychain`.  
- **Cron job can’t read secrets**: move to launchd user agent or 1Password CLI export at runtime.  
- **Accidentally committed secrets**: rotate immediately; purge history if public.

---

## 9) Recap
```plaintext
Prefer Keychain or vault → reference secrets in .env (no plaintext) → retrieve at runtime via keyring or CLI → scope per env/user → rotate and audit
```

**Next**: For cloud-managed secrets, integrate with `022_AZURE_AUTOMATION.md` (Key Vault) or `023_AWS_AUTOMATION.md` (Secrets Manager/SSM). Use schedulers in `026_TASK_SCHEDULING.md` to run jobs that fetch secrets securely at start-up.