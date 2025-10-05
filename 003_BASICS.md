

# Python Basics — updated Oct 5, 2025

Beginner‑friendly refresher and cheatsheet. Learn core syntax and patterns, then build a tiny CLI tool. Assumes macOS + zsh. Use with `002_SETUP.md` for environment setup.

---

**Note on Conventions:**
- Run examples inside an activated virtual environment.  
- Lines without a prompt are output.  
- Use Python 3.12+ or 3.13 where possible.

---

## Table of Contents
- [0) Quick Start: Run Python](#0-quick-start-run-python)
- [1) Printing and f‑strings](#1-printing-and-f-strings)
- [2) Variables and Basic Types](#2-variables-and-basic-types)
- [3) Operators](#3-operators)
- [4) Collections: list, tuple, set, dict](#4-collections-list-tuple-set-dict)
- [5) Slicing and Comprehensions](#5-slicing-and-comprehensions)
- [6) Conditionals](#6-conditionals)
- [7) Loops](#7-loops)
- [8) Functions](#8-functions)
- [9) Modules and Imports](#9-modules-and-imports)
- [10) Files and Paths](#10-files-and-paths)
- [11) Errors and Exceptions](#11-errors-and-exceptions)
- [12) Classes and Data Classes](#12-classes-and-data-classes)
- [13) Type Hints](#13-type-hints)
- [14) CLI App: argparse + .env](#14-cli-app-argparse--env)
- [15) Testing with pytest (very short)](#15-testing-with-pytest-very-short)
- [16) Next Steps](#16-next-steps)

---

## 0) Quick Start: Run Python
**What**: Start the REPL or run a file.  
**Why**: Fast feedback while learning.
```sh
python  # REPL; exit() to quit
python - <<'PY'
print("Hello from stdin")
PY
python basics_demo.py  # run a file
```

---

## 1) Printing and f‑strings
**What**: Output text.  
**Why**: Debugging and UX.  
```python
print("Hello")
name = "Ned"
print(f"Hello, {name}! Upper={name.upper()} len={len(name)}")
```
Output:
```
Hello
Hello, Ned! Upper=NED len=3
```

---

## 2) Variables and Basic Types
**What**: Bind names to values.  
**Why**: All state lives in variables.
```python
n = 42              # int
pi = 3.14159        # float
s = "text"          # str
flag = True         # bool
n = n + 1           # rebound
```
`type(x)` reveals the runtime type. Strings use single or double quotes; triple quotes for multi‑line.

---

## 3) Operators
**Arithmetic**: `+ - * / // % **`  
**Compare**: `== != < <= > >=` return `bool`  
**Logic**: `and or not`  
**Membership**: `in`, `not in`  
**Identity**: `is`, `is not` (object identity, not value equality)  
```python
3 / 2     # 1.5
3 // 2    # 1  (floor division)
2 ** 10   # 1024
"py" * 3  # 'pypypy'
```

---

## 4) Collections: list, tuple, set, dict
**What**: Built‑ins for grouping data.  
**Why**: 90% of day‑to‑day work.
```python
# list: ordered, mutable
nums = [1, 2, 3]
nums.append(4)
nums[0] = 99

# tuple: ordered, immutable
pt = (10, 20)

# set: unique items, no order
s = {1, 2, 2, 3}
assert s == {1, 2, 3}

# dict: key → value mapping
user = {"name": "Ned", "role": "IT"}
user["active"] = True
```
Keys in dicts are usually strings or immutables. Use `.get(key, default)` to avoid `KeyError`.

---

## 5) Slicing and Comprehensions
**What**: Extract parts and build new collections.  
**Why**: Concise transformations.
```python
nums = [0,1,2,3,4,5]
nums[1:4]      # [1,2,3]
nums[:3]       # [0,1,2]
nums[-2:]      # [4,5]
nums[::2]      # [0,2,4]

# comprehensions
squares = [n*n for n in nums]
evens = [n for n in nums if n % 2 == 0]
set_upper = {c.upper() for c in "abca"}
index_map = {i: v for i, v in enumerate(["a","b"]) }
```

---

## 6) Conditionals
```python
x = 10
if x > 5:
    print("big")
elif x == 5:
    print("five")
else:
    print("small")

# Ternary
size = "big" if x > 5 else "small"
```

---

## 7) Loops
```python
# for over iterables
for n in [1,2,3]:
    print(n)

# range
for i in range(3):
    print(i)  # 0,1,2

# while
count = 0
while count < 3:
    count += 1

# loop helpers
for i, val in enumerate(["a","b"]):
    print(i, val)
for key, val in {"a":1,"b":2}.items():
    print(key, val)
```

---

## 8) Functions
**What**: Reusable blocks.  
**Why**: Structure and testability.
```python
def greet(name: str = "World") -> str:
    return f"Hello, {name}!"

print(greet())
print(greet("Ned"))

# *args / **kwargs
def add_all(*nums: int) -> int:
    return sum(nums)

# Inner functions and closures
def make_multiplier(k: int):
    def mul(x: int) -> int: return k * x
    return mul
```
Docstrings:
```python
def area(r: float) -> float:
    """Return area of a circle with radius r."""
    import math
    return math.pi * r * r
```

---

## 9) Modules and Imports
**What**: Organize code across files.  
**Why**: Maintainability.
```
project/
├── src/
│   ├── __init__.py
│   └── util.py
└── app.py
```
`src/util.py`:
```python
def shout(s: str) -> str:
    return s.upper()
```
`app.py` run from project root:
```python
from src.util import shout
print(shout("hello"))
```
> Use `python -m package.module` to run modules by path.

---

## 10) Files and Paths
```python
from pathlib import Path
p = Path("data.txt")
p.write_text("line1\n")
print(p.read_text())

# iterate lines
with p.open() as f:
    for line in f:
        print(line.strip())
```
`pathlib` is preferred over `os.path`.

---

## 11) Errors and Exceptions
```python
try:
    1 / 0
except ZeroDivisionError as e:
    print("cannot divide by zero", e)
else:
    print("no error")
finally:
    print("always runs")

# Raise your own
raise ValueError("bad input")
```

---

## 12) Classes and Data Classes
```python
class Counter:
    def __init__(self, start: int = 0):
        self.value = start
    def inc(self):
        self.value += 1

c = Counter(); c.inc(); print(c.value)
```
Dataclasses reduce boilerplate:
```python
from dataclasses import dataclass
@dataclass
class User:
    name: str
    active: bool = True

u = User("Ned")
print(u.name, u.active)
```

---

## 13) Type Hints
**What**: Optional static types.  
**Why**: Better tooling, fewer bugs.
```python
from typing import List, Dict, Optional
names: List[str] = ["a", "b"]
index: Dict[str, int] = {"a": 0}
maybe: Optional[int] = None
```
Run a checker:
```sh
pip install mypy
mypy your_project/
```

---

## 14) CLI App: argparse + .env
**Goal**: Small tool that greets a user and hits a web API if requested.

`cli.py`:
```python
from argparse import ArgumentParser
from dotenv import load_dotenv
from pathlib import Path
import os

load_dotenv()  # loads variables from .env in cwd

parser = ArgumentParser(description="Greet and show options")
parser.add_argument("name", nargs="?", default=os.getenv("ENV_NAME", "World"))
parser.add_argument("--upper", action="store_true", help="Uppercase output")
args = parser.parse_args()

msg = f"Hello, {args.name}!"
if args.upper:
    msg = msg.upper()
print(msg)
```
Run:
```sh
echo "ENV_NAME=Ned" > .env
python cli.py
python cli.py Alice --upper
```
Expected:
```
Hello, Ned!
HELLO, ALICE!
```

> Networking example (optional):
```python
# add to cli.py after parsing
if os.getenv("DEMO_HTTP", "0") == "1":
    import requests
    print("GET https://httpbin.org/get ...")
    print(requests.get("https://httpbin.org/get").status_code)
```
```sh
pip install requests
DEMO_HTTP=1 python cli.py
```

---

## 15) Testing with pytest (very short)
**What**: Validate behavior.  
**Why**: Prevent regressions.
```sh
pip install pytest
mkdir -p tests
```
`tests/test_greet.py`:
```python
from cli import msg if False else None  # placeholder to avoid import side effects

def test_math():
    assert 2 + 2 == 4
```
Run:
```sh
pytest -q
```
> Real tests import functions instead of executing scripts at import time. Refactor CLI logic into functions for testability.

---

## 16) Next Steps
- Read `002_SETUP.md` for venvs, `.env`, requirements, and pyenv.
- Learn `logging`, `datetime`, `itertools`, `functools`, and `collections`.
- Package a project with `pyproject.toml` and `build`.
- Add linters: `ruff`, formatter: `black`, and types: `mypy`.

---

### Recap Flow
```plaintext
Print → Variables → Collections → Control Flow → Functions → Imports → Files → Exceptions → Classes → Types → CLI → Tests
```
