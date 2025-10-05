

# Running System Commands with `subprocess` — updated Oct 5, 2025

Practical guide to executing system commands from Python using `subprocess`. Covers safe command construction, capturing output, return codes, errors, timeouts, streaming output, environment and working directory control, pipelines, and asynchronous patterns with `asyncio`. Use with `002_SETUP.md` for environment setup.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Examples prefer **list argv** over shell strings for safety.

---

## Table of Contents
- [0) When to use subprocess](#0-when-to-use-subprocess)
- [1) Basics: run a command and check status](#1-basics-run-a-command-and-check-status)
- [2) Capturing stdout/stderr as text](#2-capturing-stdoutstderr-as-text)
- [3) Handling errors and non‑zero exits](#3-handling-errors-and-non-zero-exits)
- [4) Timeouts](#4-timeouts)
- [5) Working directory and environment variables](#5-working-directory-and-environment-variables)
- [6) Streaming output live](#6-streaming-output-live)
- [7) Sending input to a process](#7-sending-input-to-a-process)
- [8) Pipelines without the shell](#8-pipelines-without-the-shell)
- [9) Asynchronous subprocess with asyncio](#9-asynchronous-subprocess-with-asyncio)
- [10) Shell usage and security](#10-shell-usage-and-security)
- [11) Signals, termination, and cleanup](#11-signals-termination-and-cleanup)
- [12) Cross‑platform tips](#12-cross-platform-tips)
- [13) Troubleshooting](#13-troubleshooting)
- [14) Recap](#14-recap)

---

## 0) When to use subprocess
**What**: Bridge Python with existing CLI tools.  
**Why**: Reuse mature utilities, avoid re‑implementing logic, integrate with system scripts.

Rule: prefer Python libraries when stable, but call CLIs when they are the source of truth (git, rsync, ffmpeg, openssl, kubectl, az, aws).

---

## 1) Basics: run a command and check status
Use `subprocess.run` with a list of arguments.
```python
import subprocess
res = subprocess.run(["echo", "hello"], capture_output=True, text=True)
print(res.returncode)   # 0 means success
print(res.stdout.strip())
```
`res` is a `CompletedProcess` with `args`, `returncode`, `stdout`, `stderr`.

---

## 2) Capturing stdout/stderr as text
**What**: Get command output for logging or parsing.  
**Why**: Many CLIs print results to stdout.
```python
import subprocess
res = subprocess.run(["uname", "-a"], capture_output=True, text=True)
print(res.stdout)
print(res.stderr)
```
Tips:
- Use `text=True` (or `encoding="utf-8"`) to decode bytes.  
- For large outputs, prefer streaming (Section 6) or `Popen`.

---

## 3) Handling errors and non‑zero exits
**What**: Fail fast on errors or branch on return codes.
```python
import subprocess
try:
    subprocess.run(["ls", "/no/such"], check=True)
except subprocess.CalledProcessError as e:
    print("cmd failed", e.returncode, e.cmd)
```
Without `check=True`, you must test `returncode` yourself. Log `stderr` for diagnostics.

---

## 4) Timeouts
**What**: Avoid hung processes.
```python
import subprocess
try:
    subprocess.run(["sleep", "5"], timeout=1)
except subprocess.TimeoutExpired as e:
    print("timed out", e)
```
Combine with retries/backoff as needed.

---

## 5) Working directory and environment variables
**What**: Control process context.
```python
import os, subprocess
from pathlib import Path

# working directory
res = subprocess.run(["ls"], cwd=str(Path.home()))

# custom env overlay
env = {**os.environ, "ENV_NAME": "Ned"}
res = subprocess.run(["python", "-c", "import os; print(os.getenv('ENV_NAME'))"], env=env, text=True, capture_output=True)
print(res.stdout.strip())  # Ned
```

---

## 6) Streaming output live
**What**: See output as it happens (long‑running tools, progress, tails).
```python
import subprocess, sys

with subprocess.Popen(["ping", "-c", "3", "127.0.0.1"], stdout=subprocess.PIPE, stderr=subprocess.STDOUT, text=True) as p:
    for line in p.stdout:  # iterates as lines arrive
        sys.stdout.write(line)
    code = p.wait()
    print("\nexit:", code)
```
`Popen` gives fine‑grained control over pipes and lifecycle.

---

## 7) Sending input to a process
**What**: Provide stdin to interactive or filter programs.
```python
import subprocess
p = subprocess.Popen(["grep", "foo"], stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True)
out, _ = p.communicate("foo\nbar\n")
print(out)
```
Always call `communicate()` to flush pipes and avoid deadlocks.

---

## 8) Pipelines without the shell
**What**: Connect processes safely without `shell=True`.
```python
import subprocess
p1 = subprocess.Popen(["echo", "alpha\nbeta\ngamma"], stdout=subprocess.PIPE, text=True)
p2 = subprocess.Popen(["grep", "a"], stdin=p1.stdout, stdout=subprocess.PIPE, text=True)
p1.stdout.close()  # allow p1 to receive SIGPIPE if p2 exits
out, _ = p2.communicate()
print(out)
```
For multi‑stage pipelines, chain `Popen` objects similarly.

---

## 9) Asynchronous subprocess with asyncio
**What**: Run subprocesses without blocking the event loop; manage many tasks concurrently.
```python
import asyncio

async def run_cmd():
    proc = await asyncio.create_subprocess_exec(
        "python", "-c", "import time; print('start'); time.sleep(1); print('end')",
        stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE,
    )
    out, err = await proc.communicate()
    print(out.decode())
    return proc.returncode

asyncio.run(run_cmd())
```
Use `asyncio.wait_for(task, timeout)` to bound time; schedule multiple subprocesses with `gather`.

---

## 10) Shell usage and security
Prefer **argv lists**: `subprocess.run(["cmd", "arg"])`. Avoid `shell=True` unless you need shell features (globbing, `|`, `&&`, redirection). If you must:
```python
import subprocess
subprocess.run("echo $HOME && ls *.md", shell=True, executable="/bin/zsh")
```
Security: never pass untrusted input into shell strings. Use `shlex.quote` if composing.

---

## 11) Signals, termination, and cleanup
**What**: Stop processes cleanly; avoid zombies.
```python
import subprocess, signal, time
p = subprocess.Popen(["sleep", "30"])  # long task
try:
    time.sleep(1)
    p.terminate()            # SIGTERM
    try:
        p.wait(timeout=5)
    except subprocess.TimeoutExpired:
        p.kill()             # SIGKILL
finally:
    code = p.poll()
    print("exit:", code)
```
On POSIX you can send specific signals with `p.send_signal(signal.SIGINT)`.

---

## 12) Cross‑platform tips
- Prefer Python commands over platform‑specific tools (`python -c`, `pathlib`).  
- Use `shutil.which("tool")` to detect command availability.  
- Avoid hardcoded locations; rely on PATH or config.  
- Newlines differ; read in **text mode** to normalize, or handle bytes explicitly.

---

## 13) Troubleshooting
- **`FileNotFoundError`**: command not in PATH; use absolute path or `shutil.which`.  
- **Hangs**: forgetting `communicate()` while capturing both stdout and stderr.  
- **Garbled Unicode**: set `text=True` and `encoding="utf-8"`.  
- **Permission denied**: script not executable; run via interpreter or `chmod +x`.  
- **Timeouts**: wrap with `timeout=` and handle `TimeoutExpired`.  
- **Return code > 0**: log `stderr`; with `check=True` catch `CalledProcessError`.

---

## 14) Recap
```plaintext
Prefer argv lists → run → capture stdout/stderr → check return codes → add timeouts → stream when large/long → wire pipelines with Popen → use asyncio for concurrency → avoid shell=True unless necessary → handle signals and cleanup
```

**Next**: For shell/Python hybrids see `024_SHELL_INTERACTION.md`. For monitoring and alerting around command results see `025_MONITORING_SCRIPTS.md`. For scheduling recurring jobs see `026_TASK_SCHEDULING.md`. 