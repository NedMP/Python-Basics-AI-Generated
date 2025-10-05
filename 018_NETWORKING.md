

# Python Networking Basics — updated Oct 5, 2025

Practical primer for network scripting with Python. Covers sockets, simple TCP/UDP clients and servers, pinging hosts, DNS lookups, basic port checks, HTTP health probes, and small diagnostic tools. Use with `002_SETUP.md` for environment setup.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- You have basic CLI access on target machines.  
- Examples avoid admin privileges where possible.

---

## Table of Contents
- [0) Networking modules you will use](#0-networking-modules-you-will-use)
- [1) Quick connectivity checks (ping, DNS, HTTP)](#1-quick-connectivity-checks-ping-dns-http)
- [2) TCP client and server](#2-tcp-client-and-server)
- [3) UDP datagrams](#3-udp-datagrams)
- [4) Timeouts, retries, and errors](#4-timeouts-retries-and-errors)
- [5) Simple port scanner](#5-simple-port-scanner)
- [6) Small diagnostic script](#6-small-diagnostic-script)
- [7) Async networking with asyncio](#7-async-networking-with-asyncio)
- [8) Useful libraries and tools](#8-useful-libraries-and-tools)
- [9) Troubleshooting](#9-troubleshooting)
- [10) Recap](#10-recap)

---

## 0) Networking modules you will use
**What**: Core stdlib pieces for network tasks.  
**Why**: Reduce dependencies and keep scripts portable.
- `socket` — low-level TCP/UDP.  
- `ssl` — TLS wrapping for sockets.  
- `asyncio` — async I/O and servers.  
- `subprocess` — call system tools like `ping`/`traceroute`.  
- `socket.getaddrinfo`, `ipaddress` — resolution and address handling.  
- `http.client` or `requests/httpx` — HTTP probing.

---

## 1) Quick connectivity checks (ping, DNS, HTTP)
**What**: Fast checks to separate DNS vs network vs service issues.

### 1.1 DNS lookup
```python
import socket
print(socket.getaddrinfo("example.com", None)[0][4])  # (ip, port) tuple or just IP
```

### 1.2 Ping host (no root) via system `ping`
On macOS, raw ICMP requires privileges. Use `ping` via `subprocess` for portability.
```python
import subprocess, shlex

def ping(host: str, count: int = 2, timeout: int = 2) -> bool:
    cmd = f"ping -c {count} -W {timeout} {shlex.quote(host)}"  # macOS: -W timeout seconds
    res = subprocess.run(cmd, shell=True, capture_output=True, text=True)
    return res.returncode == 0

print(ping("1.1.1.1"))
```

### 1.3 HTTP health check
```python
import requests

r = requests.get("https://example.com/health", timeout=5)
print(r.status_code, r.elapsed.total_seconds())
```

---

## 2) TCP client and server
**What**: Connect to TCP services or expose a small service for testing.

### 2.1 TCP client
```python
import socket

HOST, PORT = "example.com", 80
with socket.create_connection((HOST, PORT), timeout=3) as s:
    s.sendall(b"HEAD / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")
    data = s.recv(1024)
    print(data.decode(errors="replace"))
```

### 2.2 Minimal TCP echo server
```python
import socket

HOST, PORT = "127.0.0.1", 5000
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as srv:
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind((HOST, PORT))
    srv.listen(5)
    print(f"listening on {HOST}:{PORT}")
    while True:
        conn, addr = srv.accept()
        with conn:
            print("client:", addr)
            while chunk := conn.recv(1024):
                conn.sendall(chunk)
```

---

## 3) UDP datagrams
**What**: Fire‑and‑forget or request/response with low overhead.

### 3.1 UDP client
```python
import socket

HOST, PORT = "127.0.0.1", 6000
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as s:
    s.sendto(b"hello", (HOST, PORT))
    s.settimeout(2)
    try:
        data, addr = s.recvfrom(1024)
        print("from", addr, data)
    except socket.timeout:
        print("no reply")
```

### 3.2 UDP echo server
```python
import socket

HOST, PORT = "127.0.0.1", 6000
with socket.socket(socket.AF_INET, socket.SOCK_DGRAM) as srv:
    srv.bind((HOST, PORT))
    print(f"listening on {HOST}:{PORT}")
    while True:
        data, addr = srv.recvfrom(2048)
        print("from", addr, data)
        srv.sendto(data, addr)
```

---

## 4) Timeouts, retries, and errors
**What**: Avoid hangs and give stable behavior on flaky networks.
```python
import socket, time

def tcp_check(host: str, port: int, retries=3, timeout=2.0) -> bool:
    for i in range(retries):
        try:
            with socket.create_connection((host, port), timeout=timeout):
                return True
        except (socket.timeout, OSError):
            time.sleep(0.2 * (2**i))  # backoff
    return False

print(tcp_check("example.com", 443))
```

---

## 5) Simple port scanner
**What**: Test a small set of ports quickly for reachability.
```python
import socket

def scan(host: str, ports: list[int], timeout=0.5):
    open_ports = []
    for p in ports:
        try:
            with socket.create_connection((host, p), timeout=timeout):
                open_ports.append(p)
        except OSError:
            pass
    return open_ports

print(scan("scanme.nmap.org", [22, 80, 443, 8080, 3306]))
```
Notes:
- Only scan hosts you own or have permission for.  
- This checks connect() success, not full service correctness.

---

## 6) Small diagnostic script
**What**: Combine DNS + ping + TCP checks to narrow failures.
```python
# file: net_diag.py
import socket, subprocess, shlex, sys

def ping(host: str) -> bool:
    cmd = f"ping -c 2 -W 2 {shlex.quote(host)}"
    return subprocess.run(cmd, shell=True).returncode == 0

host = sys.argv[1] if len(sys.argv) > 1 else "example.com"

print("DNS:")
try:
    infos = socket.getaddrinfo(host, None)
    addrs = sorted({i[4][0] for i in infos})
    for a in addrs:
        print(" ", a)
except Exception as e:
    print("  lookup failed:", e)

print("PING:", "ok" if ping(host) else "fail")
print("TCP 80:", "open" if ping(host) and __import__('socket').create_connection((host, 80), 2) or False else "closed")
```
Run:
```sh
python net_diag.py example.com
```

---

## 7) Async networking with asyncio
**What**: Handle many connections concurrently with low overhead.

### 7.1 Async TCP server
```python
import asyncio

async def handle(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    data = await reader.readline()
    writer.write(data.upper())
    await writer.drain()
    writer.close()
    await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle, "127.0.0.1", 7000)
    async with server:
        await server.serve_forever()

asyncio.run(main())
```

### 7.2 Async TCP client
```python
import asyncio

async def main():
    reader, writer = await asyncio.open_connection("127.0.0.1", 7000)
    writer.write(b"hello\n"); await writer.drain()
    data = await reader.read(100)
    print(data)
    writer.close(); await writer.wait_closed()

asyncio.run(main())
```

---

## 8) Useful libraries and tools
- **requests / httpx** — user‑friendly HTTP clients; `httpx` supports async.  
- **dnspython** — DNS queries and records.  
- **scapy** — packet crafting and sniffing (requires privileges).  
- **pythonping** — ICMP from Python; may require sudo on macOS.  
- **psutil** — inspect connections and interfaces.  
- System tools: `ping`, `traceroute`, `nc` (netcat), `telnet`, `curl`, `dig`.

---

## 9) Troubleshooting
- **Permission denied for ICMP**: macOS requires root for raw sockets; use system `ping` via `subprocess` or run with `sudo`.  
- **Name resolves but connect fails**: firewall or service down; test alternate ports.  
- **Hangs**: set socket timeouts; never rely on defaults.  
- **IPv6 vs IPv4**: force `AF_INET`/`AF_INET6` or try both from `getaddrinfo`.  
- **TLS failures**: wrap sockets with `ssl.create_default_context().wrap_socket(...)` or use `requests`.

---

## 10) Recap
```plaintext
Resolve DNS → ping or HTTP probe → test TCP/UDP with timeouts → scan select ports → combine checks into a tiny diagnostic → scale with asyncio when handling many hosts
```

**Next**: For subprocess patterns see `017_SUBPROCESS.md`. For scheduling recurring checks see `026_TASK_SCHEDULING.md`. For alerting, integrate with `031_NOTIFICATION_AUTOMATION.md`. 