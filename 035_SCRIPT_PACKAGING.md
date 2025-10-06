

# Packaging Scripts into Reusable CLI Tools — updated Oct 6, 2025

Turn Python scripts into installable, versioned **CLI tools**. This guide covers command‑line interfaces with **argparse** or **click**, packaging with **setuptools** or **poetry**, creating console **entry points**, editable installs for dev, dependency pinning, and publishing options. Use with `002_SETUP.md` for venv setup.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- You want `mypkg` to install a `mytool` command anywhere on PATH.

**Install**
```sh
pip install build twine
# optional frameworks
pip install click
# optional modern packaging
pip install poetry
```

---

## Table of Contents
- [0) Project layout](#0-project-layout)
- [1) CLI with argparse (stdlib)](#1-cli-with-argparse-stdlib)
- [2) CLI with click (ergonomic)](#2-cli-with-click-ergonomic)
- [3) Packaging with setuptools (pyproject)](#3-packaging-with-setuptools-pyproject)
- [4) Packaging with poetry](#4-packaging-with-poetry)
- [5) Editable installs for development](#5-editable-installs-for-development)
- [6) Versioning and dependencies](#6-versioning-and-dependencies)
- [7) Building wheels and sdists](#7-building-wheels-and-sdists)
- [8) Publishing and private indexes](#8-publishing-and-private-indexes)
- [9) Distributing without publish (artifacts, zipapps)](#9-distributing-without-publish-artifacts-zipapps)
- [10) Testing installed CLIs](#10-testing-installed-clis)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Project layout
**What**: Standard layout so packaging tools find code and metadata.  
**Why**: Clean imports, easy tests, reliable builds.
```
myproj/
├── pyproject.toml
├── README.md
├── src/
│   └── mypkg/
│       ├── __init__.py
│       └── cli.py
└── tests/
    └── test_cli.py
```
Tips:
- Use **`src/` layout** to avoid importing from the working tree.  
- Keep the CLI in `mypkg/cli.py` and expose via an entry point.

---

## 1) CLI with argparse (stdlib)
**What**: Build a zero‑dependency CLI with flags, subcommands, and help.  
**Why**: Ships with Python; good default.
```python
# file: src/mypkg/cli.py
from __future__ import annotations
import argparse

__version__ = "0.1.0"

def main(argv: list[str] | None = None) -> int:
    p = argparse.ArgumentParser(prog="mytool", description="Demo tool")
    p.add_argument("name", nargs="?", default="World", help="Name to greet")
    p.add_argument("-v", "--version", action="version", version=__version__)

    sub = p.add_subparsers(dest="cmd")
    ping = sub.add_parser("ping", help="Check connectivity")
    ping.add_argument("host", help="Host to ping")

    args = p.parse_args(argv)
    if args.cmd == "ping":
        print(f"Pinging {args.host} ... ok")
        return 0
    print(f"Hello, {args.name}!")
    return 0

if __name__ == "__main__":
    raise SystemExit(main())
```
Run locally (before packaging):
```sh
python -m mypkg.cli --version
python -m mypkg.cli Ned
python -m mypkg.cli ping 1.1.1.1
```

---

## 2) CLI with click (ergonomic)
**What**: Decorator‑based CLI with colors, prompts, and nested commands.  
**Why**: Faster to build complex CLIs.
```python
# file: src/mypkg/cli.py
from __future__ import annotations
import click

__version__ = "0.1.0"

@click.group(help="Demo CLI")
@click.version_option(__version__, prog_name="mytool")
def cli():
    pass

@cli.command()
@click.argument("name", required=False, default="World")
def greet(name: str):
    click.echo(f"Hello, {name}!")

@cli.command()
@click.argument("host")
def ping(host: str):
    click.echo(f"Pinging {host} ... ok")

def main() -> None:
    cli()  # click auto-dispatch

if __name__ == "__main__":
    main()
```

---

## 3) Packaging with setuptools (pyproject)
**What**: Define metadata, dependencies, and entry points using **PEP 621** in `pyproject.toml`.  
**Why**: Modern, tool‑agnostic configuration.
```toml
# file: pyproject.toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "mypkg"
version = "0.1.0"
description = "Demo packaged CLI"
readme = "README.md"
requires-python = ">=3.12"
license = {text = "MIT"}
authors = [{ name = "Ned Perkins" }]
dependencies = [
  # "click>=8.1,<9"   # uncomment if using click
]

[project.urls]
Homepage = "https://example.com"

[project.scripts]
mytool = "mypkg.cli:main"

[tool.setuptools]
package-dir = {"" = "src"}

[tool.setuptools.packages.find]
where = ["src"]
```
Build and install locally:
```sh
pip install -e .          # editable dev install
python -m build           # creates dist/*.whl and *.tar.gz
pip install dist/mypkg-0.1.0-py3-none-any.whl
mytool --version
```

---

## 4) Packaging with poetry
**What**: All‑in‑one dependency and build tool.  
**Why**: Simpler workflows, lockfile, and publishing UX.
```sh
poetry new myproj --src
cd myproj
poetry add click
```
`pyproject.toml` highlights:
```toml
[tool.poetry]
name = "mypkg"
version = "0.1.0"
description = "Demo packaged CLI"
authors = ["Ned Perkins"]

[tool.poetry.dependencies]
python = ">=3.12,<3.14"
click = "^8.1"

[tool.poetry.scripts]
mytool = "mypkg.cli:main"
```
Build and install:
```sh
poetry build
pip install dist/mypkg-0.1.0-py3-none-any.whl
mytool greet Ned
```
Tip: `poetry run mytool` executes in Poetry’s virtualenv.

---

## 5) Editable installs for development
**What**: Install your package in a way that imports live code from your working tree.  
**Why**: Immediate changes without rebuild.
```sh
pip install -e .
# now edits under src/mypkg are reflected instantly
```
Uninstall:
```sh
pip uninstall mypkg
```

---

## 6) Versioning and dependencies
**What**: Keep releases traceable and compatible.  
**Why**: Avoid breakage for users.
- Single source of truth: put `__version__` in code or use dynamic versioning from VCS (e.g., `setuptools-scm`).  
- Use **compatible release** pins: `click>=8.1,<9`.  
- Maintain `requirements.txt` for apps, not for libraries; libraries should declare deps in `pyproject.toml`.

Dynamic versioning example:
```toml
[build-system]
requires = ["setuptools>=68", "wheel", "setuptools-scm"]

[tool.setuptools_scm]
version_scheme = "post-release"
local_scheme = "node-and-date"
```

---

## 7) Building wheels and sdists
**What**: Produce artifacts users can install.  
**Why**: Wheels install fast and avoid build steps on user machines.
```sh
python -m build  # dist/*.whl (wheel) and *.tar.gz (sdist)
```
Validate:
```sh
pipx run twine check dist/*
```

---

## 8) Publishing and private indexes
**What**: Distribute packages to users.  
**Why**: Easy installs via `pip install mypkg`.

### PyPI (public)
```sh
python -m twine upload dist/*
```

### Private index (e.g., company Artifactory)
```sh
pip install --index-url https://pypi.example.com/simple mypkg
```
Store credentials securely (see `033_KEYCHAIN_SECRETS.md`).

---

## 9) Distributing without publish (artifacts, zipapps)
**What**: Ship directly without a package index.
- Share the **wheel** (`.whl`) or sdist and `pip install <file.whl>`.  
- Build a **zipapp** single‑file executable:
```sh
python -m zipapp src -m "mypkg.cli:main" -o mytool.pyz
python mytool.pyz --help
```
Note: Zipapps still require a Python interpreter.

---

## 10) Testing installed CLIs
**What**: Verify user experience after installation.  
**Why**: Catch packaging mistakes early.
```sh
mytool --help
mytool greet World
which mytool
python -c "import mypkg, sys; print(mypkg.__version__)"
```
For automated tests, use `pytest` + `subprocess.run` to invoke `mytool` and assert exit codes and output.

---

## 11) Troubleshooting
- **`mytool: command not found`**: not on PATH or entry point misconfigured. Check `pip show mypkg` and reinstall.  
- **Imports work but CLI fails**: `[project.scripts]` target wrong; must be `module:function`.  
- **Editable install doesn’t reflect changes**: using non‑src layout or installed in different env. Verify `which python` and `pip -V`.  
- **Click not found**: forgot to declare dependency in `pyproject.toml`.  
- **Wheel builds with wrong files**: ensure `package-dir` and `find.where` point to `src`.

---

## 12) Recap
```plaintext
Design CLI (argparse/click) → set src/ layout → define pyproject with entry point → editable install → build wheel → test CLI → publish or ship artifact
```

**Next**: Add logging (`016_LOGGING.md`), config files (`021_JSON_YAML_TOML.md`), and scheduling (`026_TASK_SCHEDULING.md`). For binary‑like distribution, consider PyInstaller or uv/pex in a future section.