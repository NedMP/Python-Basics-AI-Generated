

# Automated Testing with `unittest` and `pytest` — updated Oct 6, 2025

Write fast, reliable tests for Python scripts and packages. This guide covers **unittest** (stdlib) and **pytest** (ergonomic), including test discovery, **fixtures**, **parametrization**, **mocking**, **temporary files**, **coverage**, and **CI** integration. Use with `002_SETUP.md` for venv setup and project structure.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- `src/` layout with code under `src/` and tests under `tests/`.

**Install**
```sh
# stdlib unittest requires no install
pip install pytest pytest-cov coverage
```

---

## Table of Contents
- [0) Test layout and discovery](#0-test-layout-and-discovery)
- [1) `unittest` quick start](#1-unittest-quick-start)
- [2) `pytest` quick start](#2-pytest-quick-start)
- [3) Fixtures and temporary resources](#3-fixtures-and-temporary-resources)
- [4) Parametrized tests](#4-parametrized-tests)
- [5) Mocking I/O, HTTP, and time](#5-mocking-io-http-and-time)
- [6) Coverage: what’s tested?](#6-coverage-whats-tested)
- [7) Test data and sample files](#7-test-data-and-sample-files)
- [8) Running in CI (GitHub Actions)](#8-running-in-ci-github-actions)
- [9) Patterns, tips, and troubleshooting](#9-patterns-tips-and-troubleshooting)
- [10) Recap](#10-recap)

---

## 0) Test layout and discovery
**What**: Organize tests for automatic discovery and clean imports.  
**Why**: Predictable structure cuts friction and flakiness.
```
myproj/
├── pyproject.toml
├── src/
│   └── mypkg/
│       ├── __init__.py
│       └── mathy.py
└── tests/
    ├── test_mathy_unittest.py
    └── test_mathy_pytest.py
```
Discovery rules:
- `unittest`: files named `test*.py` or classes subclassing `unittest.TestCase`.  
- `pytest`: files named `test_*.py` or `*_test.py` with plain functions or classes.

---

## 1) `unittest` quick start
**What**: Standard library framework with xUnit style classes and assertions.  
**Why**: Zero extra dependencies and stable API.
```python
# file: src/mypkg/mathy.py
from __future__ import annotations

def add(a: int, b: int) -> int:
    return a + b
```
```python
# file: tests/test_mathy_unittest.py
import unittest
from mypkg.mathy import add

class TestMathy(unittest.TestCase):
    def test_add(self):
        self.assertEqual(add(2, 3), 5)

if __name__ == '__main__':
    unittest.main()
```
Run:
```sh
python -m unittest -v
```
Common asserts: `assertEqual`, `assertTrue`, `assertRaises`, `assertAlmostEqual`.

---

## 2) `pytest` quick start
**What**: Function‑style tests, better failure diffs, fixtures, and plugins.  
**Why**: Less boilerplate and powerful features.
```python
# file: tests/test_mathy_pytest.py
from mypkg.mathy import add

def test_add_basic():
    assert add(2, 3) == 5
```
Run:
```sh
pytest -q
```
Useful flags: `-k <expr>` select tests, `-x` stop on first failure, `-vv` verbose, `-s` show prints.

---

## 3) Fixtures and temporary resources
**What**: Create and clean up resources around tests.  
**Why**: Isolated and deterministic tests.

### 3.1 `unittest` setup/teardown
```python
class TestWithSetup(unittest.TestCase):
    def setUp(self):
        self.tmpdir = tempfile.TemporaryDirectory()
    def tearDown(self):
        self.tmpdir.cleanup()
```

### 3.2 `pytest` fixtures
```python
# file: tests/conftest.py
import pytest, tempfile

@pytest.fixture
def tmpdir_path():
    with tempfile.TemporaryDirectory() as d:
        yield d  # auto-cleanup on exit
```
Use:
```python
def test_writes_file(tmpdir_path):
    path = Path(tmpdir_path)/"out.txt"
    path.write_text("ok\n", encoding="utf-8")
    assert path.read_text() == "ok\n"
```
Built-ins: `tmp_path`, `capsys`, `monkeypatch`.

---

## 4) Parametrized tests
**What**: Run the same test with multiple inputs/outputs.  
**Why**: More coverage with less code.

### 4.1 `unittest` subTests
```python
class TestAdd(unittest.TestCase):
    def test_add_cases(self):
        cases = [(1,2,3),(0,0,0),(-1,1,0)]
        for a,b,exp in cases:
            with self.subTest(a=a,b=b):
                self.assertEqual(add(a,b), exp)
```

### 4.2 `pytest.mark.parametrize`
```python
import pytest
@pytest.mark.parametrize("a,b,exp", [(1,2,3),(0,0,0),(-1,1,0)])
def test_add_param(a,b,exp):
    assert add(a,b) == exp
```

---

## 5) Mocking I/O, HTTP, and time
**What**: Replace slow or side‑effectful calls with fakes.  
**Why**: Tests run fast and deterministically.

### 5.1 Stdlib `unittest.mock`
```python
from unittest.mock import patch, Mock

# mock a function in its import location
with patch("mypkg.mathy.add", return_value=42) as fake:
    assert fake() == 42
```

### 5.2 Mock HTTP with `requests`
```python
# file: src/mypkg/net.py
import requests

def fetch_status(url: str) -> int:
    return requests.get(url, timeout=5).status_code
```
```python
# file: tests/test_net.py
from unittest.mock import patch, Mock
from mypkg.net import fetch_status

def test_fetch_status_ok():
    with patch("mypkg.net.requests.get") as get:
        get.return_value = Mock(status_code=200)
        assert fetch_status("https://example.com") == 200
```

### 5.3 Time and environment
```python
from unittest.mock import patch
import os, time

def test_time_and_env(monkeypatch):  # pytest example
    monkeypatch.setenv("ENV_NAME", "Ned")
    with patch("time.time", return_value=1234567890):
        assert int(time.time()) == 1234567890
        assert os.getenv("ENV_NAME") == "Ned"
```

---

## 6) Coverage: what’s tested?
**What**: Measure executed lines to find gaps.  
**Why**: Prevents untested critical paths.
```sh
pytest --cov=mypkg --cov-report=term-missing
# HTML report
pytest --cov=mypkg --cov-report=html:htmlcov && open htmlcov/index.html
```
`.coveragerc` example:
```ini
[run]
branch = True
source = mypkg

[report]
omit =
    */.venv/*
    */tests/*
show_missing = True
```

---

## 7) Test data and sample files
**What**: Keep fixtures and sample inputs organized.  
**Why**: Reusable and readable tests.
```
tests/
├── data/
│   ├── small.csv
│   └── config.json
└── test_parser.py
```
Load with `Path(__file__).parent / "data" / "small.csv"` to avoid CWD issues.

---

## 8) Running in CI (GitHub Actions)
**What**: Run tests automatically on push/PR.  
**Why**: Catches regressions early.
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: macos-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.13' }
      - run: pip install -e .
      - run: pip install pytest pytest-cov
      - run: pytest --cov=mypkg --cov-report=term-missing
```
Cache deps for speed with `actions/cache` keyed on `pyproject.toml` or `requirements.txt`.

---

## 9) Patterns, tips, and troubleshooting
- **Fast tests first**: avoid network and disk; mock them.  
- **Isolate state**: use fixtures, temp dirs, and `monkeypatch`.  
- **Deterministic RNG**: seed `random.seed(0)` in a fixture.  
- **Mark slow or integration tests**: `pytest -m "not slow"`.  
- **Flaky tests**: add retries via `pytest-rerunfailures`, but fix root causes.  
- **Import errors**: ensure `src/` layout and editable install `pip install -e .`.  
- **Naming**: consistent `test_*.py` keeps discovery simple.  
- **Debugging**: `pytest -k name -vv -s`, use `pdb` with `pytest --pdb`.

---

## 10) Recap
```plaintext
Adopt src/ + tests/ layout → choose unittest or pytest → add fixtures and params → mock I/O and HTTP → measure coverage → wire into CI → keep tests fast and isolated
```

**Next**: Combine with `035_SCRIPT_PACKAGING.md` to package your app, use `016_LOGGING.md` for structured logs in tests, and `031_NOTIFICATION_AUTOMATION.md` to notify on CI failures.