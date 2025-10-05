# Python + venv on macOS (Homebrew) — updated Oct 5, 2025

This guide takes you from a clean macOS install to a functional, isolated Python environment using modern tools. It covers Python installation, virtual environments, `.env` management, dependency handling, and version control with `pyenv`.

---

**Note on Terminal Conventions:**  
Commands prefixed with `$` or shown without a prompt indicate shell input. Lines without a prompt represent output. Adjust commands as necessary for your shell (e.g., `bash`, `zsh`).

---

## Table of Contents

- [0) Install Homebrew (skip if installed)](#0-install-homebrew-skip-if-installed)  
- [1) Install the latest Python via Homebrew](#1-install-the-latest-python-via-homebrew)  
- [2) Create a project folder](#2-create-a-project-folder)  
- [3) Create and activate a virtual environment](#3-create-and-activate-a-virtual-environment)  
- [4) Upgrade packaging tools and install test packages](#4-upgrade-packaging-tools-and-install-test-packages)  
- [5) Create a `.env` file](#5-create-a-env-file)  
- [6) Create and test `app.py`](#6-create-and-test-apppy)  
- [7) Freeze and restore dependencies](#7-freeze-and-restore-dependencies)  
- [8) Recommended project structure](#8-recommended-project-structure)  
- [9) Common .gitignore](#9-common-gitignore)  
- [10) Environment management tips](#10-environment-management-tips)  
- [10.5) Test the full setup](#105-test-the-full-setup)  
- [11) Troubleshooting](#11-troubleshooting)  
- [12) Python Version Management and Modern Tooling](#12-python-version-management-and-modern-tooling)  

---

## 0) Install Homebrew (skip if installed)

Homebrew is a popular package manager for macOS that makes it easy to install, update, and manage software from the command line. Using Homebrew ensures you get the latest versions of tools and libraries without interfering with the system defaults.

```sh
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
Add Homebrew to your PATH if it isn’t already.
- Apple Silicon:
  ```sh
  echo 'eval "$($(brew --prefix)/bin/brew shellenv)"' >> ~/.zprofile && eval "$($(brew --prefix)/bin/brew shellenv)"
  ```
- Intel Macs use the same command. `brew --prefix` resolves automatically.

Verifying the installation confirms that Homebrew is ready to use and properly configured in your shell environment.
```sh
brew --version
```

---

## 1) Install the latest Python via Homebrew

Installing Python via Homebrew gives you a fresh, up-to-date version of Python that is separate from the macOS system Python. This avoids conflicts and ensures you have access to the latest language features and security updates.

**Note:** If you plan to work with multiple Python versions or need more control over Python versions, you can skip ahead to Section 12.1 (pyenv setup) before continuing with this step.

```sh
brew update
brew install python
```
Check version and location:
```sh
python3 --version
which python3
```
Expected output: `/opt/homebrew/bin/python3` (Apple Silicon) or `/usr/local/bin/python3` (Intel). If you see `/usr/bin/python3`, your PATH is wrong.

Fix PATH:
```sh
echo 'export PATH="$(brew --prefix)/bin:$PATH"' >> ~/.zprofile
exec "$SHELL"
```
This step ensures your shell finds the Homebrew-installed Python before the system default.

---

## 2) Create a project folder

Creating a dedicated project folder helps you organize your code, dependencies, and configuration in one place. This makes it easier to manage multiple projects without conflicts and keeps your work tidy and reproducible.

```sh
mkdir -p ~/Code_Stuff/hello-venv && cd ~/Code_Stuff/hello-venv
```

---

## 3) Create and activate a virtual environment

A virtual environment is an isolated Python environment that lets you install packages without affecting your global Python installation. This isolation prevents version conflicts between projects and keeps dependencies self-contained.

```sh
python3 -m venv .venv
source .venv/bin/activate
```
Your prompt shows `(.venv)` when active, for example:
```
(.venv) user@MacBook-Pro hello-venv %
```
This indicates the virtual environment is currently active.

**Note:** The virtual environment must be reactivated in each new terminal session by running `source .venv/bin/activate` again.

Inside the virtual environment, the commands `python` and `pip` refer to the isolated environment’s executables, whereas outside the venv you typically use `python3` and `pip3`. This means inside the venv, simply running `python` or `pip` will use the correct versions.

Check interpreter:
```sh
which python
python -V
```
Deactivate:
```sh
deactivate
```
Activating the virtual environment temporarily changes your shell’s Python to the isolated one inside `.venv`. Deactivating returns you to the system Python.

---

## 4) Upgrade packaging tools and install test packages

`pip` is Python’s package installer, `setuptools` helps with packaging Python projects, and `wheel` enables building and installing binary packages. Upgrading these tools ensures you have the latest features and bug fixes, which improves installation reliability.

```sh
pip install --upgrade pip setuptools wheel
pip install python-dotenv rich
```
- `python-dotenv` loads environment variables from `.env`.
- `rich` verifies package installs and enables styled output.

To demonstrate installing and using another package, try installing `requests` and confirm it works:
```sh
pip install requests
python -c "import requests; print(requests.__version__)"
```
This simple test confirms that you can install and import packages correctly in your virtual environment.

---

## 5) Create a `.env` file

A `.env` file stores configuration values and secrets (like API keys or usernames) outside your code. This helps keep your code clean and secure by not hardcoding sensitive information directly in your scripts.

Place the `.env` file in the same directory as your `app.py` script to ensure it loads correctly.

```
ENV_NAME=Ned
```
Change the name as desired.

---

## 6) Create and test `app.py`

The `app.py` script demonstrates how to use `python-dotenv` to load environment variables from the `.env` file. It reads the `ENV_NAME` variable and prints a personalized greeting. This shows how environment variables can configure your application without changing code.

```python
from dotenv import load_dotenv
import os

load_dotenv()
name = os.getenv("ENV_NAME", "World")
print(f"Hello, {name}!")
```
Run it:
```sh
python app.py
```
Expected output:
```
Hello, Ned!
```

To confirm packages, modify it:
```python
from rich import print
from dotenv import load_dotenv
import os

load_dotenv()
name = os.getenv("ENV_NAME", "World")
print(f"[bold green]Hello, {name}![/bold green]")
```
---

## 7) Freeze and restore dependencies

Freezing your requirements means capturing the exact versions of all installed packages into a `requirements.txt` file. This allows you or others to recreate the same environment later, ensuring consistent behavior across machines and deployments.

```sh
pip freeze > requirements.txt
```
To recreate:
```sh
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Note:**  
- `requirements.txt` typically contains the packages needed for the application to run.  
- `requirements-dev.txt` is often used to list additional packages needed for development and testing (e.g., linters, test frameworks). Separating these helps keep production installs lean.

---

## 8) Recommended project structure

A clean project structure separates code, configuration, and tests into logical folders. This modular organization makes your project easier to navigate, maintain, and scale. Keeping code in `src/` and tests in `tests/` is a common best practice.

```
hello-venv/
├── .venv/
├── .env
├── app.py
├── requirements.txt
├── requirements-dev.txt
├── src/
│   ├── __init__.py
│   └── main.py
└── tests/
    └── test_main.py
```
Always run scripts from the project root to ensure paths resolve correctly.

**Why this structure helps:**  
It keeps environment files and virtual environments separate from source code and tests, reducing clutter and accidental commits. Organizing code under `src/` improves import behavior and clarity, while `tests/` isolates testing code, making maintenance and scaling easier.

---

## 9) Common .gitignore

A `.gitignore` file tells Git which files or folders to ignore when tracking changes. This prevents temporary files, sensitive data, or build artifacts from cluttering your version control history or accidentally being shared.

```
.venv/
.env
__pycache__/
*.pyc
.DS_Store
```

---

## 10) Environment management tips

Managing environment variables carefully is important for reproducibility and security. A `.env.example` file with placeholder values helps collaborators set up their own `.env` files correctly. Tools like VS Code’s dotenv extension or `direnv` automate loading `.env` files for convenience.

- Create a `.env.example` file with placeholder values.
- Use VS Code’s dotenv or **direnv** to auto-load `.env`.
- Export manually if needed:
  ```sh
  export ENV_NAME=Ned
  python app.py
  ```

---

## 10.5) Test the full setup

Before moving on, verify your environment is correctly configured by following these steps:

1. Activate your virtual environment:
   ```sh
   source .venv/bin/activate
   ```
2. Run the `app.py` script and confirm the greeting appears as expected:
   ```sh
   python app.py
   ```
3. Check installed packages and versions:
   ```sh
   pip list
   ```
4. Freeze requirements and verify the `requirements.txt` is updated:
   ```sh
   pip freeze > requirements.txt
   cat requirements.txt
   ```
5. Deactivate the virtual environment and reactivate it in a new terminal to confirm persistence:
   ```sh
   deactivate
   source .venv/bin/activate
   ```
6. Optionally, test installing a new package (e.g., `requests`) and importing it:
   ```sh
   pip install requests
   python -c "import requests; print(requests.__version__)"
   ```

Completing these checks ensures your Python environment, virtual environment, and project setup are all functioning as intended.

---

## 11) Troubleshooting

Common issues often stem from misconfigured paths, versions, or SSL certificates.

- **Wrong Python:** If `which python` points outside `.venv`, your virtual environment isn’t active. Activating it ensures you use the correct interpreter and packages.
- **PATH:** Homebrew’s bin directory must come before `/usr/bin` so your shell finds the right Python and tools first.
- **SSL errors:** Pip relies on SSL certificates; upgrading `certifi` fixes many connection problems.
- **Proxy issues:** If behind a proxy, setting `HTTPS_PROXY` and `HTTP_PROXY` ensures pip can access the internet.

---

### Recap Flow Diagram

```plaintext
Clean macOS install
        │
        ▼
Install Homebrew (Section 0)
        │
        ▼
Install Python via Homebrew (Section 1)
        │
        ▼
(Optional) Setup pyenv for multiple versions (Section 12.1)
        │
        ▼
Create project folder (Section 2)
        │
        ▼
Create and activate virtual environment (Section 3)
        │
        ▼
Upgrade pip, setuptools, wheel, install packages (Section 4)
        │
        ▼
Create .env file (Section 5)
        │
        ▼
Create and test app.py (Section 6)
        │
        ▼
Freeze dependencies (Section 7)
        │
        ▼
Organize project structure (Section 8)
        │
        ▼
Test full setup (Section 10.5)
        │
        ▼
Troubleshoot if needed (Section 11)
        │
        ▼
(Optional) Use pyenv and modern tooling (Section 12)
```

---

## 12) Python Version Management and Modern Tooling

Using tools like `pyenv` and modern Python tooling is optional but highly recommended, especially if you manage multiple projects or require different Python versions. These tools help keep your development environment clean, consistent, and easily reproducible across machines.

## 12.1) Python Version Management with pyenv

`pyenv` allows you to install, manage, and switch between multiple Python versions without interfering with macOS system Python. It works by shimming your shell to point to user-defined Python interpreters.

### Why use pyenv
- Keeps system Python untouched (important for macOS stability).
- Lets you install multiple versions (e.g. 3.9, 3.10, 3.13) and switch globally or per project.
- Integrates with `venv` or `pyenv-virtualenv` for isolated environments.

---

### Install pyenv
```sh
brew install pyenv
```
Then add it to your shell startup file.
For **zsh** (default macOS shell):
```sh
echo 'eval "$(pyenv init --path)"' >> ~/.zprofile
echo 'eval "$(pyenv init -)"' >> ~/.zshrc
```
Reload the shell:
```sh
exec "$SHELL"
```

Verify installation:
```sh
pyenv --version
```

---

### Install Python versions
List all available versions:
```sh
pyenv install --list | grep 3.
```
Install one (or more):
```sh
pyenv install 3.13.0
pyenv install 3.12.4
```

Set a global default version:
```sh
pyenv global 3.13.0
```
Confirm it:
```sh
python --version
which python
```
You should see something like:
```
Python 3.13.0
~/.pyenv/shims/python
```

---

### Local (per‑project) versions
Inside a project folder:
```sh
cd ~/Code_Stuff/hello-venv
pyenv local 3.13.0
```
This writes a `.python-version` file in the folder, automatically selecting that interpreter when you `cd` into it.

To check what’s active:
```sh
pyenv version
```

---

### Using pyenv with venv
Once your desired version is active:
```sh
python -m venv .venv
source .venv/bin/activate
```
The venv now uses the Python version selected by pyenv.

You can verify this with:
```sh
python -V
which python
```
The path should be something like:
```
~/.pyenv/versions/3.13.0/envs/hello-venv/bin/python
```

---

### Optional: pyenv‑virtualenv
Install if you want pyenv to handle virtual environments too:
```sh
brew install pyenv-virtualenv
```
Then add initialization to your shell config:
```sh
echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.zshrc
exec "$SHELL"
```
Create and activate a virtualenv through pyenv:
```sh
pyenv virtualenv 3.13.0 hello-env
pyenv activate hello-env
```
Deactivate with:
```sh
pyenv deactivate
```
List environments:
```sh
pyenv versions
```
Delete one:
```sh
pyenv uninstall hello-env
```

---

### Troubleshooting and Tips
- **PATH order:** Ensure `pyenv` shims appear before `/usr/bin` in your PATH. Run `pyenv doctor` if installed.
- **Build dependencies:** If `pyenv install` fails, run:
  ```sh
  brew install openssl readline zlib xz
  ```
- **Updating pyenv:**
  ```sh
  brew upgrade pyenv
  ```
- **Shell not detecting new versions:** Run `exec "$SHELL"` or restart terminal.

---

### Quick reference
| Action | Command |
|---------|----------|
| List all versions | `pyenv versions` |
| List installable | `pyenv install --list` |
| Install new version | `pyenv install 3.13.0` |
| Set global default | `pyenv global 3.13.0` |
| Set local version | `pyenv local 3.13.0` |
| Create virtualenv | `pyenv virtualenv 3.13.0 myenv` |
| Activate virtualenv | `pyenv activate myenv` |
| Deactivate virtualenv | `pyenv deactivate` |
| Remove version/env | `pyenv uninstall 3.12.4` |

---

**Summary:**
`pyenv` gives you clean Python version control across projects. Combine it with `venv` or `pyenv-virtualenv` for fully isolated environments. This ensures consistent builds and reproducible results regardless of macOS system Python changes.