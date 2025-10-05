

# Python Logging Fundamentals — updated Oct 5, 2025

Practical guide to production‑grade logging in Python. Covers core concepts, logger hierarchy, levels, handlers, formatters, rotating files, time‑based rotation, JSON structured logs, context data, per‑module patterns, configuration via `dictConfig`, testing, and troubleshooting. Use with `002_SETUP.md` for environment setup.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Examples are copy‑pasteable. Replace paths as needed.  
- Optional extras: `rich`, `python-json-logger`, or `structlog` for advanced formatting.

---

## Table of Contents
- [0) Why logging instead of print](#0-why-logging-instead-of-print)
- [1) Logging model: loggers, handlers, formatters, levels](#1-logging-model-loggers-handlers-formatters-levels)
- [2) Minimal, correct setup](#2-minimal-correct-setup)
- [3) Per‑module logger pattern](#3-per-module-logger-pattern)
- [4) Handlers: console, file, rotating, timed rotating](#4-handlers-console-file-rotating-timed-rotating)
- [5) JSON structured logging](#5-json-structured-logging)
- [6) Context: extra fields, adapters, contextualvars](#6-context-extra-fields-adapters-contextualvars)
- [7) Configuration with dictConfig](#7-configuration-with-dictconfig)
- [8) Tuning third‑party loggers](#8-tuning-third-party-loggers)
- [9) Logging best practices](#9-logging-best-practices)
- [10) Testing and capturing logs](#10-testing-and-capturing-logs)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Why logging instead of print
**What**: Logging gives levels, routing, formatting, rotation, and structured output.  
**Why**: You can change verbosity and destinations without editing code. Messages include timestamps, module names, and context.

---

## 1) Logging model: loggers, handlers, formatters, levels
**Loggers** create records and form a hierarchy (`app.web` child of `app`).  
**Handlers** send records to destinations (console, file, syslog, HTTP).  
**Formatters** turn records into strings or JSON.  
**Levels** control importance: `DEBUG < INFO < WARNING < ERROR < CRITICAL`.

Flow: `logger.debug(...) → handlers → formatter → output`.

---

## 2) Minimal, correct setup
**What**: One line for quick scripts; a small config for apps.
```python
# quick script
import logging
logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(name)s: %(message)s")
logging.info("hello")
```
Better function for small apps:
```python
# logging_setup.py
import logging

def setup_logging(level: int = logging.INFO) -> None:
    fmt = "%(asctime)s %(levelname)s %(name)s [%(process)d:%(threadName)s] %(message)s"
    logging.basicConfig(level=level, format=fmt)
```
Use:
```python
from logging_setup import setup_logging
setup_logging()
```

---

## 3) Per‑module logger pattern
**What**: Each module gets its own named logger.  
**Why**: Enables focused filtering and consistent prefixes.
```python
# in any module
import logging
log = logging.getLogger(__name__)

def run():
    log.info("starting")
    try:
        1 / 0
    except ZeroDivisionError:
        log.exception("division failed")  # includes stack trace
```
Tip: Use `log.exception` inside `except` to capture traceback automatically.

---

## 4) Handlers: console, file, rotating, timed rotating
**What**: Route logs to multiple targets.
```python
import logging
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler

log = logging.getLogger("demo")
log.setLevel(logging.DEBUG)

# console
ch = logging.StreamHandler()
ch.setLevel(logging.INFO)
ch.setFormatter(logging.Formatter("%(asctime)s %(levelname)s %(name)s: %(message)s"))

# size‑based rotation (5 files × 10 MB)
rfh = RotatingFileHandler("logs/app.log", maxBytes=10*1024*1024, backupCount=5, encoding="utf-8")
rfh.setLevel(logging.DEBUG)
rfh.setFormatter(logging.Formatter("%(asctime)s %(levelname)s %(name)s %(filename)s:%(lineno)d: %(message)s"))

# time‑based rotation (midnight, keep 7 days)
trh = TimedRotatingFileHandler("logs/app_daily.log", when="midnight", interval=1, backupCount=7, encoding="utf-8", utc=False)
trh.setLevel(logging.INFO)

for h in (ch, rfh, trh):
    log.addHandler(h)

log.info("logger configured")
```
Notes:
- Create the `logs/` dir first.  
- Use `RotatingFileHandler` for bounded disk usage; `TimedRotatingFileHandler` for daily files.  
- On macOS, ensure the process has write permission to the folder.

---

## 5) JSON structured logging
**What**: Emit JSON per log line for easy parsing in ELK/Splunk/CloudWatch.  
**Why**: Machines parse JSON better than free‑form text.
```sh
pip install python-json-logger
```
```python
import logging
from pythonjsonlogger import jsonlogger

log = logging.getLogger("svc")
log.setLevel(logging.INFO)

handler = logging.StreamHandler()
fmt = jsonlogger.JsonFormatter("%(asctime)s %(levelname)s %(name)s %(message)s %(filename)s %(lineno)d %(process)d %(threadName)s")
handler.setFormatter(fmt)
log.addHandler(handler)

log.info("user login", extra={"user_id": 123, "ip": "203.0.113.9"})
```
Sample output:
```json
{"asctime": "2025-10-05 13:37:00,000", "levelname": "INFO", "name": "svc", "message": "user login", "user_id": 123, "ip": "203.0.113.9"}
```
**Alternative**: `structlog` for richer pipelines and context binding.

---

## 6) Context: extra fields, adapters, contextualvars
**What**: Attach request/job IDs or tenant info to every log line.
```python
import logging

log = logging.getLogger("api")

# 1) one‑off extra
log.info("processed", extra={"request_id": "req_abc123"})

# 2) LoggerAdapter for repeated context
class RequestAdapter(logging.LoggerAdapter):
    def process(self, msg, kwargs):
        return msg, {**kwargs, "extra": {**self.extra, **kwargs.get("extra", {})}}

req_log = RequestAdapter(log, {"request_id": "req_abc123"})
req_log.info("fetching user")
```
For async/task contexts across functions, prefer `contextvars` or `structlog` with context binding.

---

## 7) Configuration with dictConfig
**What**: Centralized config from dict/JSON/YAML.  
**Why**: Keep code clean and make behavior environment‑driven.
```python
import logging, logging.config, os, sys

LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO")

DICT_CONFIG = {
  "version": 1,
  "disable_existing_loggers": False,
  "formatters": {
    "std": {"format": "%(asctime)s %(levelname)s %(name)s: %(message)s"},
    "detail": {"format": "%(asctime)s %(levelname)s %(name)s %(filename)s:%(lineno)d: %(message)s"}
  },
  "handlers": {
    "console": {"class": "logging.StreamHandler", "level": LOG_LEVEL, "formatter": "std", "stream": "ext://sys.stdout"},
    "rot": {"class": "logging.handlers.RotatingFileHandler", "level": "DEBUG", "formatter": "detail", "filename": "logs/app.log", "maxBytes": 10485760, "backupCount": 5, "encoding": "utf-8"}
  },
  "loggers": {
    "": {"level": LOG_LEVEL, "handlers": ["console", "rot"]},  # root logger
    "urllib3": {"level": "WARNING", "handlers": ["console"], "propagate": False}
  }
}

logging.config.dictConfig(DICT_CONFIG)
log = logging.getLogger("app")
log.info("configured via dictConfig")
```
Tip: Load the dict from JSON/YAML to avoid changing code between environments.

---

## 8) Tuning third‑party loggers
**What**: Reduce noise from libraries and enable only what you need.
```python
import logging
for noisy in ("urllib3", "botocore", "azure"):  # adjust per project
    logging.getLogger(noisy).setLevel(logging.WARNING)
```
To disable propagation for a child logger:
```python
l = logging.getLogger("app.chat")
l.propagate = False
```

---

## 9) Logging best practices
- Use `__name__` loggers in each module.  
- Log **events**, not everything. Prefer INFO for lifecycle, DEBUG for diagnostics.  
- Never log secrets. Redact tokens, keys, and PII.  
- Add stable keys in JSON (`request_id`, `user_id`, `tenant`, `component`).  
- Keep line length reasonable; avoid multi‑line logs except for exceptions.  
- Use UTC timestamps in distributed systems; local time for desktop tools.  
- Bound disk usage with rotation. Monitor file sizes.  
- Prefer `exception()` in except blocks. Avoid swallowing exceptions silently.  
- Make level configurable via env (`LOG_LEVEL=DEBUG`).

---

## 10) Testing and capturing logs
**What**: Assert logs in unit tests; capture to memory.
```python
import logging
from io import StringIO

stream = StringIO()
handler = logging.StreamHandler(stream)
handler.setFormatter(logging.Formatter("%(levelname)s:%(name)s:%(message)s"))

log = logging.getLogger("test")
log.handlers.clear()
log.addHandler(handler)
log.setLevel(logging.INFO)

log.info("hi")
handler.flush()
assert "INFO:test:hi" in stream.getvalue()
```
With `pytest`, use the built‑in `caplog` fixture to assert emitted records.

---

## 11) Troubleshooting
- **Duplicate lines**: you added handlers multiple times. Guard your setup or clear handlers before re‑adding.  
- **Nothing prints**: logger level too high or no handler attached. Check root vs child levels.  
- **Timezones wrong**: set `logging.Formatter(fmt, datefmt, defaults)` and use `time.gmtime` via custom formatter or prefer JSON + downstream normalization.  
- **Slow logging**: synchronous file I/O blocking; buffer logs or offload heavy formatting.  
- **Rotating fails**: ensure directory exists and process has write permission.  
- **Stack trace missing**: use `log.exception(...)` or `exc_info=True`.

---

## 12) Recap
```plaintext
Know the pieces (logger/handler/formatter/level) → set minimal config → per‑module loggers → route to console + rotating files → add JSON structured output → attach context → centralize via dictConfig → tune third‑party noise → test and monitor
```

**Next**: Hook logs into monitoring/alerts (`025_MONITORING_SCRIPTS.md`) and ship structured logs to your platform of choice. For CLI color, add `rich`’s `RichHandler` as an optional console handler.