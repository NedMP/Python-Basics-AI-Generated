

# System Information with Python (platform, psutil) — updated Oct 6, 2025

Collect CPU, RAM, disk, process, network, and uptime details using Python’s **stdlib** (`platform`, `os`, `subprocess`) and **`psutil`**. Includes quick one‑liners, a reusable function library, and a small CLI that prints JSON or table output. Works on macOS, Linux, and Windows. Pair with `002_SETUP.md` for venv and package setup.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- Cross‑platform where possible; mac‑specific notes are called out.

**Install**
```sh
pip install psutil rich platformdirs
```

---

## Table of Contents
- [0) Why psutil + platform](#0-why-psutil--platform)
- [1) OS and hardware basics](#1-os-and-hardware-basics)
- [2) CPU info and utilization](#2-cpu-info-and-utilization)
- [3) Memory (RAM + swap)](#3-memory-ram--swap)
- [4) Disk partitions, usage, and I/O](#4-disk-partitions-usage-and-io)
- [5) Processes: list, filter, and details](#5-processes-list-filter-and-details)
- [6) Network interfaces, IPs, and I/O](#6-network-interfaces-ips-and-io)
- [7) Boot time and uptime](#7-boot-time-and-uptime)
- [8) Temperatures, fans, battery (where supported)](#8-temperatures-fans-battery-where-supported)
- [9) A mini CLI: JSON or table output](#9-a-mini-cli-json-or-table-output)
- [10) Logging and scheduling](#10-logging-and-scheduling)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Why psutil + platform
**What**: `platform` reports OS and Python build info. `psutil` provides portable system metrics for CPU, memory, disks, processes, and network.
**Why**: Cross‑platform consistency and fewer shell‑specific scripts. On macOS, `psutil` wraps many syscalls that would otherwise require `sysctl`, `top`, or `iostat` parsing.

---

## 1) OS and hardware basics
**What**: Identify OS, kernel, architecture, and Python runtime.
```python
import platform, sys
print({
    "python": sys.version.split()[0],
    "implementation": platform.python_implementation(),
    "system": platform.system(),
    "release": platform.release(),
    "version": platform.version(),
    "machine": platform.machine(),
    "processor": platform.processor(),
})
```
macOS hardware via `sysctl` (optional):
```python
import subprocess, json
hw = subprocess.run(["sysctl","-n","machdep.cpu.brand_string"], capture_output=True, text=True).stdout.strip()
print({"cpu_brand": hw})
```

---

## 2) CPU info and utilization
**What**: Count cores and measure usage.
```python
import psutil, time
print({"physical_cores": psutil.cpu_count(logical=False), "logical_cores": psutil.cpu_count()})
print({"freq_mhz": getattr(psutil.cpu_freq(), "current", None)})
print("per‑cpu % over 1s sample:")
print(psutil.cpu_percent(interval=1, percpu=True))
```
Tip: Use a small `interval` to get an averaged snapshot rather than an instantaneous 0.0.

---

## 3) Memory (RAM + swap)
**What**: Total/used/available RAM and swap.
```python
import psutil
vm = psutil.virtual_memory()
swap = psutil.swap_memory()
print({
  "ram_total": vm.total, "ram_used": vm.used, "ram_free": vm.available,
  "swap_total": swap.total, "swap_used": swap.used
})
```
Convert bytes for readability:
```python
def human(n:int) -> str:
    units = ["B","KB","MB","GB","TB"]
    i = 0
    while n >= 1024 and i < len(units)-1:
        n //= 1024; i += 1
    return f"{n} {units[i]}"
```

---

## 4) Disk partitions, usage, and I/O
**What**: List mounted filesystems, capacity, and read/write counters.
```python
import psutil
for p in psutil.disk_partitions(all=False):
    try:
        u = psutil.disk_usage(p.mountpoint)
        print(p.device, p.mountpoint, u.total, u.used, u.free, u.percent)
    except PermissionError:
        pass

io = psutil.disk_io_counters(perdisk=True)
print({k: {"read_mb": v.read_bytes//1_000_000, "write_mb": v.write_bytes//1_000_000} for k,v in io.items()})
```
Note: On macOS, APFS containers may appear as multiple logical volumes.

---

## 5) Processes: list, filter, and details
**What**: Iterate processes safely and filter by name or resource usage.
```python
import psutil
for p in psutil.process_iter(["pid","name","username","cpu_percent","memory_info"]):
    try:
        info = p.info
        if info["cpu_percent"] and info["cpu_percent"] > 50:
            print(info)
    except (psutil.NoSuchProcess, psutil.AccessDenied):
        continue
```
Single process details:
```python
p = psutil.Process(psutil.Process().pid)
print({
  "pid": p.pid, "name": p.name(), "exe": p.exe(),
  "cmdline": p.cmdline(), "create_time": p.create_time(),
  "cpu": p.cpu_percent(interval=0.2),
  "rss": p.memory_info().rss
})
```

---

## 6) Network interfaces, IPs, and I/O
**What**: Interface addresses and per‑NIC traffic counters.
```python
import psutil, socket
addrs = psutil.net_if_addrs(); stats = psutil.net_if_stats(); io = psutil.net_io_counters(pernic=True)
for name, a in addrs.items():
    ips = [s.address for s in a if s.family in (socket.AF_INET, socket.AF_INET6)]
    up = stats[name].isup if name in stats else None
    txrx = io.get(name)
    print(name, {"ips": ips, "up": up, "tx_mb": (txrx.bytes_sent//1_000_000) if txrx else 0, "rx_mb": (txrx.bytes_recv//1_000_000) if txrx else 0})
```

---

## 7) Boot time and uptime
**What**: System startup timestamp and elapsed time.
```python
import psutil, time
boot = psutil.boot_time()
uptime = time.time() - boot
print({"boot_time": boot, "uptime_seconds": int(uptime)})
```
macOS uptime via `sysctl` (alt): `sysctl -n kern.boottime`.

---

## 8) Temperatures, fans, battery (where supported)
**What**: Hardware sensors when available.
```python
import psutil
print(getattr(psutil, "sensors_temperatures", lambda: {})())
print(getattr(psutil, "sensors_fans", lambda: {})())
print(getattr(psutil, "sensors_battery", lambda: None)())
```
Note: Sensors are limited on macOS without extra permissions or tools; values may be empty.

---

## 9) A mini CLI: JSON or table output
**What**: A single tool to print a snapshot for humans or machines.
```python
# file: sysinfo.py
from __future__ import annotations
import json, time, socket, platform, sys
import psutil

def gather() -> dict:
    vm = psutil.virtual_memory(); swap = psutil.swap_memory()
    cpu = {
        "physical": psutil.cpu_count(logical=False),
        "logical": psutil.cpu_count(),
        "percent": psutil.cpu_percent(interval=0.5),
    }
    disks = {}
    for p in psutil.disk_partitions(all=False):
        try:
            u = psutil.disk_usage(p.mountpoint)
            disks[p.mountpoint] = {"total": u.total, "used": u.used, "free": u.free, "percent": u.percent}
        except PermissionError:
            continue
    net = {}
    nio = psutil.net_io_counters(pernic=True)
    for name, st in psutil.net_if_stats().items():
        addrs = [a.address for a in psutil.net_if_addrs()[name] if a.family in (socket.AF_INET, socket.AF_INET6)]
        io = nio.get(name)
        net[name] = {"up": st.isup, "speed": st.speed, "ips": addrs,
                     "tx": getattr(io, "bytes_sent", 0), "rx": getattr(io, "bytes_recv", 0)}
    return {
        "host": platform.node(),
        "os": {"system": platform.system(), "release": platform.release(), "machine": platform.machine()},
        "python": sys.version.split()[0],
        "boot_time": psutil.boot_time(),
        "uptime": int(time.time() - psutil.boot_time()),
        "cpu": cpu,
        "memory": {"total": vm.total, "available": vm.available, "used": vm.used},
        "swap": {"total": swap.total, "used": swap.used},
        "disks": disks,
        "net": net,
        "top_procs": [p.info for p in sorted(psutil.process_iter(["pid","name","cpu_percent","memory_info"]), key=lambda x: (x.info.get("cpu_percent") or 0), reverse=True)[:5]],
    }

if __name__ == "__main__":
    import argparse
    ap = argparse.ArgumentParser(description="System info snapshot")
    ap.add_argument("--json", action="store_true", help="output JSON")
    args = ap.parse_args()
    data = gather()
    if args.json:
        print(json.dumps(data, indent=2))
    else:
        from rich import print
        from rich.table import Table
        t = Table(title=f"System info: {data['host']}")
        t.add_column("Key"); t.add_column("Value")
        t.add_row("OS", f"{data['os']['system']} {data['os']['release']} ({data['os']['machine']})")
        t.add_row("Python", data['python'])
        t.add_row("Uptime", str(data['uptime']) + "s")
        t.add_row("CPU", f"{data['cpu']['physical']} phys / {data['cpu']['logical']} log @ {data['cpu']['percent']}%")
        vm = data['memory']; t.add_row("RAM", f"used {vm['used']}/{vm['total']}")
        print(t)
```
Run:
```sh
python sysinfo.py            # pretty table
python sysinfo.py --json     # machine-readable
```

---

## 10) Logging and scheduling
**What**: Periodically capture snapshots for auditing or baselining.
- Use `016_LOGGING.md` for structured logs.  
- Schedule with `026_TASK_SCHEDULING.md` or a `launchd` plist.  
- Write JSON to a rotating file and send alerts via `031_NOTIFICATION_AUTOMATION.md` if thresholds are exceeded.

---

## 11) Troubleshooting
- **Permission errors**: some process or sensor data is restricted; run with appropriate rights or skip fields.  
- **Zero CPU%**: call `cpu_percent(interval=0.5)` to allow sampling.  
- **Missing temps on macOS**: expected; Apple sensors are not universally exposed.  
- **Docker containers**: metrics reflect the host namespace unless limited by cgroups; consider container‑aware tools.  
- **IPv6 only/none**: handle empty lists for interface addresses.

---

## 12) Recap
```plaintext
platform → OS basics
psutil → CPU, RAM, disks, processes, net
Add CLI → JSON/table → schedule and log as needed
```

**Next**: Feed outputs to `011_GRAPHS.md` for visualization, alert via `031_NOTIFICATION_AUTOMATION.md`, package as a CLI with `035_SCRIPT_PACKAGING.md`, and containerize with `036_DOCKER_AUTOMATION.md`. 