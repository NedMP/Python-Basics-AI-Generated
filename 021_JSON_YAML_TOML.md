

# Working with JSON, YAML, and TOML — updated Oct 6, 2025

Practical guide for reading, writing, validating, and merging common configuration formats in Python: **JSON**, **YAML**, and **TOML**. Covers round‑tripping with comments, schema validation, env var interpolation, deep merges, and safe defaults. Use with `002_SETUP.md` for environment setup.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Keep secrets out of config files unless encrypted; prefer `.env` + vaults.

**Install**
```sh
pip install pyyaml tomlkit jsonschema pydantic
# optional: ruamel.yaml for comment-preserving YAML
pip install ruamel.yaml
```

---

## Table of Contents
- [0) Choose the right format](#0-choose-the-right-format)
- [1) JSON: fast and strict](#1-json-fast-and-strict)
- [2) YAML: human-friendly with pitfalls](#2-yaml-human-friendly-with-pitfalls)
- [3) TOML: configuration with types and comments](#3-toml-configuration-with-types-and-comments)
- [4) Env var interpolation](#4-env-var-interpolation)
- [5) Validating configs](#5-validating-configs)
- [6) Merging strategies](#6-merging-strategies)
- [7) Round‑trip editing while preserving comments](#7-round-trip-editing-while-preserving-comments)
- [8) Tips for structure and defaults](#8-tips-for-structure-and-defaults)
- [9) Troubleshooting](#9-troubleshooting)
- [10) Recap](#10-recap)

---

## 0) Choose the right format
**What**: Match the file format to the task.  
**Why**: Reduces friction and parsing errors.
- **JSON**: best for machine‑to‑machine, fastest in stdlib, no comments.  
- **YAML**: human‑oriented, supports comments and anchors, indentation‑sensitive.  
- **TOML**: structured config with first‑class types, comments, good for apps and tooling.

---

## 1) JSON: fast and strict
**What**: Stdlib `json` provides reliable parsing and emitting.
```python
import json
from pathlib import Path

cfg_path = Path("config.json")
# read
data = json.loads(cfg_path.read_text(encoding="utf-8"))
# or streaming
with cfg_path.open("r", encoding="utf-8") as f:
    data = json.load(f)

# write (pretty)
cfg_path.write_text(json.dumps(data, indent=2, sort_keys=True), encoding="utf-8")
```
Notes:
- No comments per spec. Strip `//` or `#` with a preprocessor if needed.  
- Use `parse_float`, `parse_int` hooks for numeric control if required.

---

## 2) YAML: human-friendly with pitfalls
**What**: Use `pyyaml` with **safe** loaders/dumpers.
```python
import yaml
from pathlib import Path
cfg = yaml.safe_load(Path("config.yaml").read_text(encoding="utf-8"))
# write with safe dumper
Path("config.yaml").write_text(yaml.safe_dump(cfg, sort_keys=False), encoding="utf-8")
```
Anchors and overrides (advanced):
```yaml
# config.yaml
base: &base
  host: api.example.com
  timeout: 10

prod:
  <<: *base
  timeout: 20
```
Cautions:
- Indentation matters; tabs are invalid.  
- Avoid `yaml.load` without Loader. Use `safe_load` to prevent arbitrary object construction.

---

## 3) TOML: configuration with types and comments
**What**: Use `tomlkit` for round‑trip with comments preserved.
```python
from pathlib import Path
import tomlkit

text = Path("config.toml").read_text(encoding="utf-8")
doc = tomlkit.parse(text)
# access
timeout = int(doc["http"]["timeout"])  # tomlkit types are wrappers; cast if needed
# modify
doc["http"]["timeout"] = 15
Path("config.toml").write_text(tomlkit.dumps(doc), encoding="utf-8")
```
Example TOML:
```toml
[http]
base_url = "https://api.example.com"
# seconds
timeout = 10
```
Notes:
- For read‑only parsing, `tomllib` (stdlib in 3.11+) works but can’t write. Use `tomlkit` to write while keeping comments.

---

## 4) Env var interpolation
**What**: Replace `${VAR}` placeholders from the environment.
```python
import os, re

pattern = re.compile(r"\$\{([A-Z0-9_]+)\}")

def interpolate(obj):
    if isinstance(obj, dict):
        return {k: interpolate(v) for k, v in obj.items()}
    if isinstance(obj, list):
        return [interpolate(v) for v in obj]
    if isinstance(obj, str):
        return pattern.sub(lambda m: os.getenv(m.group(1), m.group(0)), obj)
    return obj

# usage after loading 'data' dict from any format
# os.environ["API_KEY"] = "secret"
# data = interpolate(data)
```
Tip: Keep secrets in `.env` and load via `python-dotenv` before interpolation.

---

## 5) Validating configs
**What**: Ensure structure and types match expectations.

### 5.1 Pydantic models (Pythonic validation)
```python
from pydantic import BaseModel, Field, ValidationError
from typing import Optional

class HTTP(BaseModel):
    base_url: str
    timeout: int = Field(10, ge=1, le=120)

class AppConfig(BaseModel):
    env: str
    http: HTTP
    feature_flag: Optional[bool] = False

try:
    cfg = AppConfig.model_validate(data)  # 'data' from JSON/YAML/TOML
except ValidationError as e:
    print(e)
```

### 5.2 JSON Schema (cross‑language)
```python
from jsonschema import validate, Draft202012Validator
schema = {
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "env": {"type": "string"},
    "http": {
      "type": "object",
      "properties": {
        "base_url": {"type": "string"},
        "timeout": {"type": "integer", "minimum": 1, "maximum": 120}
      },
      "required": ["base_url"]
    }
  },
  "required": ["env", "http"]
}
Draft202012Validator(schema).validate(data)
```

---

## 6) Merging strategies
**What**: Combine defaults, environment, and user overrides.

### 6.1 Shallow merge (Python ≥3.9)
```python
merged = defaults | env_overrides | user_cfg  # later wins
```

### 6.2 Deep merge (dict of dicts)
```python
def deep_merge(a: dict, b: dict) -> dict:
    out = a.copy()
    for k, v in b.items():
        if k in out and isinstance(out[k], dict) and isinstance(v, dict):
            out[k] = deep_merge(out[k], v)
        else:
            out[k] = v
    return out

merged = deep_merge(defaults, env_overrides)
merged = deep_merge(merged, user_cfg)
```
Order of precedence (recommended): `defaults` → `env` → `local file` → `CLI flags`.

---

## 7) Round‑trip editing while preserving comments
**What**: Keep human comments intact when modifying files programmatically.

### 7.1 YAML with ruamel.yaml
```python
from ruamel.yaml import YAML
from pathlib import Path

yaml = YAML()
yaml.preserve_quotes = True

p = Path("config.yaml")
data = yaml.load(p.read_text(encoding="utf-8"))
# modify
data["http"]["timeout"] = 15
yaml.dump(data, p.open("w", encoding="utf-8"))
```

### 7.2 TOML with tomlkit
Already preserves comments by default (see Section 3).

---

## 8) Tips for structure and defaults
- Keep **one root object** with clear sections (`http`, `auth`, `paths`).  
- Provide a `config.example.*` with sane defaults and comments.  
- Prefer **lowercase snake_case** keys for consistency.  
- Avoid mixing types for the same key across environments.  
- Document units (`timeout_seconds`) to prevent ambiguity.  
- Keep file size small; split per‑env files if necessary: `config.default.yaml`, `config.prod.yaml`.

---

## 9) Troubleshooting
- **YAML parse errors**: check spaces vs tabs; ensure proper indentation.  
- **Numbers read as strings**: cast explicitly or validate with Pydantic.  
- **Losing comments**: use `ruamel.yaml` or `tomlkit` for round‑trip.  
- **Unicode/encoding**: always read/write with `encoding="utf-8"`.  
- **Env placeholders not replaced**: confirm variables are exported or load `.env` first.  
- **Conflicting overrides**: print the final merged dict and log precedence.

---

## 10) Recap
```plaintext
Pick format → load (safe) → interpolate env → validate (Pydantic/JSON Schema) → deep‑merge defaults+overrides → write back with preserved comments when needed
```

**Next**: Feed validated config into scripts from `020_API_AUTOMATION.md` and `017_SUBPROCESS.md`. Store secrets in `.env` and keep config files in version control with a `config.example.*` template.