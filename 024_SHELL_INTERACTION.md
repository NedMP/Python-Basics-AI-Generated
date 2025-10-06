

# Python ↔ Shell Interaction — updated Oct 6, 2025

Practical guide to combining Python with shell scripting. Covers passing environment variables, quoting safely, piping and redirection, heredocs, calling external commands with `subprocess` (sync and async), capturing output, timeouts, working directories, and return-code handling. Use with `002_SETUP.md` for environment/venv and see `017_SUBPROCESS.md` for deeper subprocess patterns.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 inside a venv.  
- You know basic shell commands and want safe, repeatable automation.

---

## Table of Contents
- [0) When to use the shell vs Python](#0-when-to-use-the-shell-vs-python)
- [1) Environment variables: read, write, and pass-through](#1-environment-variables-read-write-and-pass-through)
- [2) Running commands safely (no shell)](#2-running-commands-safely-no-shell)
- [3) Piping and redirection without a shell](#3-piping-and-redirection-without-a-shell)
- [4) When `shell=True` is acceptable](#4-when-shelltrue-is-acceptable)
- [5) Heredocs from Python](#5-heredocs-from-python)
- [6) Working directory and PATH control](#6-working-directory-and-path-control)
- [7) Timeouts, exit codes, and errors](#7-timeouts-exit-codes-and-errors)
- [8) Quoting and escaping: `shlex`](#8-quoting-and-escaping-shlex)
- [9) Small utilities: blending Python + shell](#9-small-utilities-blending-python--shell)
- [10) Async subprocess with asyncio](#10-async-subprocess-with-asyncio)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) When to use the shell vs Python
**What**: Decide whether to call a CLI or write pure Python.  
**Why**: Fewer bugs and simpler maintenance.
- Use **Python libraries** for stable, well‑supported tasks (HTTP, JSON, CSV).  
- Use **CLI tools** when they are the source of truth or much faster (git, grep, sed, rsync, ffmpeg, kubectl, az, aws).  
- Wrap CLIs with **subprocess** and keep arguments as a list for safety.

---

## 1) Environment variables: read, write, and pass-through
**What**: Scripts often need configuration via environment variables.  
**Why**: Avoid hardcoding secrets and per‑host differences.

### 1.1 Read OS env in Python
```python
import os
print(os.getenv("ENV_NAME", "World"))
```

### 1.2 Set env for child process only
```python
import os, subprocess
env = {**os.environ, "API_TOKEN": "abc123"}
subprocess.run(["bash", "-lc", "echo $API_TOKEN"], env=env)
```

### 1.3 Load from .env (local dev)
```python
from dotenv import load_dotenv; load_dotenv()  # then read via os.getenv
```

---

## 2) Running commands safely (no shell)
**What**: Prefer argv lists and `shell=False`.  
**Why**: Avoid shell injection and quoting problems.
```python
import subprocess
res = subprocess.run(["ls", "-la", "/tmp"], capture_output=True, text=True, check=False)
print(res.returncode)
print(res.stdout)
```
Use `check=True` to raise on non‑zero exits.

---

## 3) Piping and redirection without a shell
**What**: Connect processes directly with pipes.
```python
import subprocess, sys
p1 = subprocess.Popen(["echo", "alpha\nbeta\ngamma"], stdout=subprocess.PIPE, text=True)
p2 = subprocess.Popen(["grep", "a"], stdin=p1.stdout, stdout=subprocess.PIPE, text=True)
p1.stdout.close()
out, _ = p2.communicate()
sys.stdout.write(out)
```
Redirect output to a file safely:
```python
from pathlib import Path
with Path("out.txt").open("w", encoding="utf-8") as f:
    subprocess.run(["date"], stdout=f, check=True)
```

---

## 4) When `shell=True` is acceptable
**What**: Use sparingly for features like globbing, `|`, `>`, `&&`, or complex expansions.  
**Why**: Convenience at the cost of security.
```python
import subprocess
cmd = "ls -1 *.md | wc -l"
subprocess.run(cmd, shell=True, executable="/bin/zsh", check=True)
```
Rules:
- Never inject untrusted input.  
- If you must interpolate, **quote** it (see Section 8).

---

## 5) Heredocs from Python
**What**: Send multi‑line scripts to a shell process.
```python
import subprocess, textwrap
script = textwrap.dedent(
    """
    cat > demo.txt <<'EOF'
    line 1
    $HOME is not expanded because of single quotes
    EOF
    sed -n '1,1p' demo.txt
    """
)
subprocess.run(["bash", "-lc", script], check=True)
```
Tip: Use **single‑quoted** heredoc markers (`<<'EOF'`) to avoid unintended variable expansion.

---

## 6) Working directory and PATH control
**What**: Run commands from a specific folder and control tool resolution.
```python
import subprocess, os
from pathlib import Path

work = Path("/tmp/demo"); work.mkdir(exist_ok=True)
subprocess.run(["git", "init"], cwd=work, check=True)

env = {**os.environ, "PATH": f"/opt/homebrew/bin:{os.environ['PATH']}"}
subprocess.run(["python", "-V"], env=env, check=True)
```

---

## 7) Timeouts, exit codes, and errors
**What**: Bound long tasks and handle failures.
```python
import subprocess
try:
    subprocess.run(["sleep", "5"], timeout=1, check=True)
except subprocess.TimeoutExpired:
    print("timeout")
except subprocess.CalledProcessError as e:
    print("failed:", e.returncode)
```
Log both `stdout` and `stderr` for diagnostics when needed.

---

## 8) Quoting and escaping: `shlex`
**What**: Quote untrusted strings for shell commands.
```python
import shlex, subprocess
filename = "notes $(rm -rf) .txt"
cmd = f"echo {shlex.quote(filename)}"
subprocess.run(cmd, shell=True, executable="/bin/zsh")
```
If you can, avoid `shell=True` and pass argv lists instead.

---

## 9) Small utilities: blending Python + shell
**What**: Use each tool where it’s strongest.

### 9.1 Python drives shell loop
```python
import subprocess
hosts = ["1.1.1.1", "8.8.8.8"]
for h in hosts:
    subprocess.run(["ping", "-c", "1", h])
```

### 9.2 Shell calls Python for logic
```sh
#!/usr/bin/env bash
# file: grep_json.sh
jq_filter='.[].id'
python - "$jq_filter" <<'PY'
import json, sys
filt = sys.argv[1]
data = json.load(sys.stdin)
# simple emulate: print keys named 'id'
if filt == '.[].id':
    if isinstance(data, list):
        for obj in data:
            print(obj.get('id'))
PY
```

### 9.3 Use `direnv` or VS Code to load `.env`
See `002_SETUP.md` Section 10 for convenience loading to keep shell clean.

---

## 10) Async subprocess with asyncio
**What**: Run many commands concurrently with backpressure.
```python
import asyncio

async def run(cmd: list[str]):
    proc = await asyncio.create_subprocess_exec(*cmd, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
    out, err = await proc.communicate()
    return proc.returncode, out.decode(), err.decode()

async def main():
    cmds = [["curl", "-sS", "https://example.com"], ["sleep", "1"]]
    results = await asyncio.gather(*(run(c) for c in cmds))
    for code, out, err in results:
        print(code, len(out))

asyncio.run(main())
```

---

## 11) Troubleshooting
- **`FileNotFoundError`**: command not in PATH; use absolute path or extend PATH in `env`.  
- **Hangs**: forgot to consume pipes; call `communicate()` or iterate line by line.  
- **Garbled Unicode**: set `text=True` and `encoding="utf-8"`.  
- **Permissions**: scripts need execute bit `chmod +x`. Run via interpreter when unsure.  
- **shell=True security**: never pass user input unquoted; prefer argv lists.

---

## 12) Recap
```plaintext
Read/env vars → run commands via argv → wire pipes/redirection directly → only use shell=True for shell features → send heredocs carefully → control cwd/PATH → add timeouts and handle codes → quote with shlex → scale with asyncio when needed
```

**Next**: Deep dive on `subprocess` in `017_SUBPROCESS.md`. Integrate outputs with `020_API_AUTOMATION.md` or feed files to `013_CSVEXCEL.md`. For scheduling recurring shell+Python jobs, see `026_TASK_SCHEDULING.md`. 