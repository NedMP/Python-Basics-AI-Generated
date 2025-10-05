

# SSH Automation with Python (Paramiko & Fabric) — updated Oct 6, 2025

Practical guide to automating SSH tasks from Python. Covers key‑based auth, running remote commands, capturing output and exit codes, timeouts, SFTP uploads/downloads, directory sync patterns, host key policy, bastion hops, SSH agent usage, multi‑host orchestration with **Fabric**, and troubleshooting. Use with `002_SETUP.md` for environment and `.env` handling.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Remote hosts run OpenSSH. You have credentials and network reachability.  
- Prefer **key‑based auth** over passwords.

---

## Table of Contents
- [0) Install packages](#0-install-packages)
- [1) SSH keys and agent](#1-ssh-keys-and-agent)
- [2) Paramiko: run a command](#2-paramiko-run-a-command)
- [3) Exit codes, timeouts, and errors](#3-exit-codes-timeouts-and-errors)
- [4) SFTP: upload, download, and perms](#4-sftp-upload-download-and-perms)
- [5) Recursively copy a folder (pattern)](#5-recursively-copy-a-folder-pattern)
- [6) Host key policy and known_hosts](#6-host-key-policy-and-known_hosts)
- [7) Bastion / jump host](#7-bastion--jump-host)
- [8) Using SSH config](#8-using-ssh-config)
- [9) Fabric: higher‑level multi‑host orchestration](#9-fabric-higher-level-multi-host-orchestration)
- [10) Parallel runs and rollups](#10-parallel-runs-and-rollups)
- [11) Security tips](#11-security-tips)
- [12) Troubleshooting](#12-troubleshooting)
- [13) Recap](#13-recap)

---

## 0) Install packages
**What**: Core SSH libs. Paramiko is the low‑level client. Fabric wraps Paramiko for ergonomic multi‑host tasks.
```sh
pip install paramiko fabric rich
```
Optional for SSH config parsing and agent helpers:
```sh
pip install sshconf
```

---

## 1) SSH keys and agent
**What**: Public‑key auth is preferred. The SSH agent lets Python use your loaded keys without handling passphrases.
```sh
# create an ed25519 key (recommended)
ssh-keygen -t ed25519 -C "ned@laptop" -f ~/.ssh/id_ed25519

# add to agent
ssh-add ~/.ssh/id_ed25519

# copy to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```
Confirm agent:
```sh
ssh-add -l
```

---

## 2) Paramiko: run a command
**What**: Connect and execute a remote command.  
**Why**: Building block for automation pipelines.
```python
# file: ssh_run.py
import paramiko

HOST, USER = "server", "user"

client = paramiko.SSHClient()
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.RejectPolicy())  # safer than AutoAdd
client.connect(HOST, username=USER, timeout=10)  # uses SSH agent keys by default

stdin, stdout, stderr = client.exec_command("uname -a")
code = stdout.channel.recv_exit_status()
print("exit:", code)
print("out:\n", stdout.read().decode())
print("err:\n", stderr.read().decode())
client.close()
```
Notes:
- If you must use a key file: `client.connect(..., key_filename=Path('~/.ssh/id_ed25519').expanduser())`.  
- Passwords: `password="..."` but avoid when possible.

---

## 3) Exit codes, timeouts, and errors
**What**: Treat non‑zero exit codes as failures; avoid hanging reads.
```python
import paramiko, socket

client = paramiko.SSHClient()
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.RejectPolicy())
try:
    client.connect("server", username="user", timeout=8, banner_timeout=8, auth_timeout=8)
    stdin, stdout, stderr = client.exec_command("systemctl is-active nginx", timeout=10)
    code = stdout.channel.recv_exit_status()
    out = stdout.read().decode(); err = stderr.read().decode()
    if code != 0:
        raise RuntimeError(f"remote failed {code}: {err or out}")
finally:
    client.close()
```
Tip: Network timeouts are separate from command timeouts; set both.

---

## 4) SFTP: upload, download, and perms
**What**: Transfer files and set permissions.
```python
# file: sftp_basic.py
import paramiko
from pathlib import Path

client = paramiko.SSHClient()
client.load_system_host_keys(); client.set_missing_host_key_policy(paramiko.RejectPolicy())
client.connect("server", username="user")

sftp = client.open_sftp()
local = Path("./report.csv")
remote = "/tmp/report.csv"
sftp.put(str(local), remote)
sftp.chmod(remote, 0o640)   # set perms
sftp.get(remote, "./report_copy.csv")
sftp.close(); client.close()
```
Create remote directories as needed:
```python
import stat
try:
    sftp.mkdir("/tmp/reports", mode=0o750)
except IOError:
    pass  # already exists
```

---

## 5) Recursively copy a folder (pattern)
**What**: Walk local paths and upload tree; useful for config bundles.
```python
# file: sftp_upload_tree.py
import os
import paramiko
from pathlib import Path

root = Path("./bundle")
client = paramiko.SSHClient(); client.load_system_host_keys(); client.set_missing_host_key_policy(paramiko.RejectPolicy())
client.connect("server", username="user")
sftp = client.open_sftp()

for path in root.rglob('*'):
    rel = path.relative_to(root)
    remote = f"/opt/app/{rel.as_posix()}"
    if path.is_dir():
        try:
            sftp.mkdir(remote)
        except IOError:
            pass
    else:
        sftp.put(str(path), remote)

sftp.close(); client.close()
```

---

## 6) Host key policy and known_hosts
**What**: Verify server identity to prevent MITM.  
**Why**: Security baseline.
- Default behavior after `load_system_host_keys()` is to trust entries in `~/.ssh/known_hosts`.  
- `RejectPolicy()` blocks unknown hosts.  
- `AutoAddPolicy()` accepts and stores unknown keys (use only for bootstrap with out‑of‑band verification).  
- To pin a key manually, load a `paramiko.hostkeys.HostKeys` file and add entries programmatically.

---

## 7) Bastion / jump host
**What**: Connect to a target through a bastion (jump) host.
```python
# file: ssh_bastion.py
import paramiko

bastion = ("bastion.example", 22)
target  = ("10.0.1.10", 22)

jump = paramiko.SSHClient(); jump.load_system_host_keys(); jump.set_missing_host_key_policy(paramiko.RejectPolicy())
jump.connect(bastion[0], port=bastion[1], username="ned")

# open a tunnel from bastion to target
transport = jump.get_transport()
chan = transport.open_channel("direct-tcpip", target, ("127.0.0.1", 0))

# use the channel as a sock for the second client
client = paramiko.SSHClient(); client.load_system_host_keys(); client.set_missing_host_key_policy(paramiko.RejectPolicy())
client.connect(target[0], sock=chan, username="ec2-user")
stdin, stdout, stderr = client.exec_command("hostname -f")
print(stdout.read().decode())
client.close(); jump.close()
```

---

## 8) Using SSH config
**What**: Reuse `~/.ssh/config` aliases (Host, User, IdentityFile, ProxyJump).
```python
import paramiko
from pathlib import Path

cfg = paramiko.SSHConfig()
with Path("~/.ssh/config").expanduser().open() as f:
    cfg.parse(f)
match = cfg.lookup("prod-web")
client = paramiko.SSHClient(); client.load_system_host_keys(); client.set_missing_host_key_policy(paramiko.RejectPolicy())
client.connect(match.get('hostname', 'prod-web'), username=match.get('user'), key_filename=match.get('identityfile'))
```
This respects your existing SSH ergonomics.

---

## 9) Fabric: higher‑level multi‑host orchestration
**What**: Fabric provides `Connection` objects and the `@task` model. Easier command execution, file ops, and env config.
```python
# file: fabfile.py
from fabric import Connection, task

HOSTS = ["web1", "web2", "web3"]

@task
def uname(c):
    c.run("uname -a")

@task
def deploy(c):
    c.put("bundle/app.tar.gz", "/opt/app/app.tar.gz")
    c.run("tar -xzf /opt/app/app.tar.gz -C /opt/app && systemctl restart app")

# run on one host:
#   fab -H web1 uname
# run on many:
#   fab -H web1,web2,web3 deploy
```
Fabric picks up SSH config and agent automatically.

---

## 10) Parallel runs and rollups
**What**: Execute across hosts concurrently and summarize results.
```python
# file: parallel.py
from concurrent.futures import ThreadPoolExecutor, as_completed
import paramiko

HOSTS = ["web1","web2","web3"]

def check(host):
    cli = paramiko.SSHClient(); cli.load_system_host_keys(); cli.set_missing_host_key_policy(paramiko.RejectPolicy())
    cli.connect(host, username="user", timeout=6)
    _, out, err = cli.exec_command("systemctl is-active nginx", timeout=8)
    code = out.channel.recv_exit_status(); cli.close()
    return host, code

with ThreadPoolExecutor(max_workers=5) as ex:
    futs = [ex.submit(check, h) for h in HOSTS]
    for fut in as_completed(futs):
        host, code = fut.result()
        print(host, "OK" if code == 0 else f"FAIL({code})")
```

---

## 11) Security tips
- Prefer **ed25519** keys. Protect private keys with passphrases.  
- Use the SSH **agent**; avoid embedding passwords/secrets in code.  
- Verify host keys and pin where possible.  
- Restrict permissions on `~/.ssh` (700) and private keys (600).  
- For `sudo` commands, configure NOPASSWD for the automation user when appropriate; otherwise expect prompts and handle them securely.

---

## 12) Troubleshooting
- **`NoValidConnectionsError`**: host/port/firewall. Test with `ssh -v host`.  
- **`AuthenticationException`**: wrong key, no agent, or server forbids your method. Confirm `ssh` works first.  
- **Hangs**: set `timeout`, `banner_timeout`, `auth_timeout`, and `exec_command(..., timeout=...)`.  
- **Host key changed**: investigate; do not blindly auto‑add. Update `known_hosts` after verification.  
- **`SFTPError`/permission denied**: path or perms wrong; check ownership and umask.  
- **Locale/PTY issues**: if commands need a TTY, request one with Fabric `c.run(cmd, pty=True)`.

---

## 13) Recap
```plaintext
Create keys → load agent → Paramiko connect → run commands with exit codes + timeouts → SFTP transfer and perms → handle host keys → bastion hops → scale with Fabric or ThreadPool → secure and verify
```

**Next**: For packaging repeated tasks into CLIs, see the planned `035_SCRIPT_PACKAGING.md`. For scheduling periodic SSH jobs, see `026_TASK_SCHEDULING.md`. For notifications, integrate with `031_NOTIFICATION_AUTOMATION.md`. 