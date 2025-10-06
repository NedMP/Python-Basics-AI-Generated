

# Network Inventory with Python — updated Oct 6, 2025

Scan subnets, discover live hosts, collect metadata (hostnames, MAC vendors, open ports, banners), and export **structured reports** (CSV/JSON). This note favors portable, low‑privilege methods by default and shows opt‑in deeper scans (Nmap, SNMP). Pair with `002_SETUP.md` for venv and general setup.

---

**Assumptions**  
- macOS or Linux, Python 3.12–3.13 in a venv.  
- You know the target CIDR (e.g., `192.168.1.0/24`).  
- Use only against networks you are authorized to scan.

**Install**
```sh
pip install python-dotenv dnspython icmplib mac-vendor-lookup tqdm rich
# optional extras
pip install python-nmap requests pysnmp
```

---

## Table of Contents
- [0) Approach and data model](#0-approach-and-data-model)
- [1) Inputs and configuration](#1-inputs-and-configuration)
- [2) Fast ping sweep (ICMP)](#2-fast-ping-sweep-icmp)
- [3) Hostname and reverse DNS](#3-hostname-and-reverse-dns)
- [4) MAC address and vendor (ARP cache)](#4-mac-address-and-vendor-arp-cache)
- [5) TCP port sampling and banners](#5-tcp-port-sampling-and-banners)
- [6) Optional: Nmap integration](#6-optional-nmap-integration)
- [7) Optional: SNMP device info](#7-optional-snmp-device-info)
- [8) Export reports (CSV/JSON)](#8-export-reports-csvjson)
- [9) Putting it together: inventory CLI](#9-putting-it-together-inventory-cli)
- [10) Scheduling and deltas](#10-scheduling-and-deltas)
- [11) Troubleshooting](#11-troubleshooting)
- [12) Recap](#12-recap)

---

## 0) Approach and data model
**What**: Define a simple, extensible record for each discovered host.  
**Why**: Consistent fields make exports, diffs, and merges trivial.

Data shape:
```json
{
  "ip": "192.168.1.10",
  "alive": true,
  "rtt_ms": 2.3,
  "hostname": "printer01.lan",
  "reverse_dns": "printer01.local",
  "mac": "AA:BB:CC:DD:EE:FF",
  "vendor": "HP Inc",
  "ports": [22,80,443],
  "banners": {"22":"OpenSSH_9.3","80":"Server: nginx"},
  "snmp": {"sysName":"PRN01","sysDescr":"HP LaserJet…"}
}
```
Store only what you collect; unknown fields can be omitted.

---

## 1) Inputs and configuration
**What**: Centralize target subnets and scan options in `.env`.  
**Why**: Reusable runs and safer defaults.

`.env` example:
```
CIDR=192.168.1.0/24
TCP_SAMPLE_PORTS=22,80,443,3389,445
TIMEOUT_MS=800
CONCURRENCY=128
```
Loader:
```python
# file: inv_config.py
from dotenv import load_dotenv; load_dotenv()
import os
from ipaddress import ip_network

CIDR = os.getenv('CIDR','192.168.1.0/24')
PORTS = [int(p) for p in os.getenv('TCP_SAMPLE_PORTS','22,80,443').split(',') if p]
TIMEOUT = int(os.getenv('TIMEOUT_MS','800'))/1000.0
CONC = int(os.getenv('CONCURRENCY','128'))
NET = list(ip_network(CIDR).hosts())
```

---

## 2) Fast ping sweep (ICMP)
**What**: Determine which IPs respond to ICMP echo and measure latency.  
**Why**: Restrict further probes to live hosts.
```python
# file: inv_ping.py
from __future__ import annotations
from icmplib import multiping
from inv_config import NET, TIMEOUT, CONC

def ping_sweep(addrs: list[str]):
    hosts = multiping([str(ip) for ip in addrs], count=1, interval=0, timeout=TIMEOUT, concurrent_tasks=CONC)
    res = {}
    for h in hosts:
        res[h.address] = {"alive": h.is_alive, "rtt_ms": round(h.avg_rtt, 2) if h.avg_rtt else None}
    return res

if __name__ == '__main__':
    print(ping_sweep(NET))
```
Notes:
- Many devices drop ICMP. Treat `alive=False` as “unknown” until TCP says otherwise.  
- `icmplib` can use raw sockets (root) or fallback methods.

---

## 3) Hostname and reverse DNS
**What**: Resolve PTR records and try forward lookups.  
**Why**: Human‑readable names aid reporting and deduping.
```python
# file: inv_dns.py
import dns.resolver, dns.reversename

def reverse_lookup(ip: str) -> dict:
    try:
        qname = dns.reversename.from_address(ip)
        ans = dns.resolver.resolve(qname, 'PTR', lifetime=1)
        ptr = str(ans[0]).rstrip('.')
        return {"reverse_dns": ptr, "hostname": ptr.split('.')[0]}
    except Exception:
        return {}
```

---

## 4) MAC address and vendor (ARP cache)
**What**: Map IP→MAC from local ARP cache and vendor via OUI DB.  
**Why**: Identify device type even without DNS.
```python
# file: inv_arp.py
import subprocess, re
from mac_vendor_lookup import MacLookup

MAC_RE = re.compile(r"([0-9a-fA-F]{2}(:|-)){5}[0-9a-fA-F]{2}")

def arp_mac(ip: str) -> dict:
    try:
        out = subprocess.check_output(["arp","-n", ip], text=True, stderr=subprocess.DEVNULL)
        m = MAC_RE.search(out)
        if not m: return {}
        mac = m.group(0).replace('-',':').lower()
        try:
            vendor = MacLookup().lookup(mac)
        except Exception:
            vendor = None
        return {"mac": mac, "vendor": vendor}
    except Exception:
        return {}
```
Tip: Populate ARP by pinging first; ARP entries expire quickly.

---

## 5) TCP port sampling and banners
**What**: Probe a small port set and read short banners.  
**Why**: Classify services without full scans.
```python
# file: inv_ports.py
from __future__ import annotations
import socket
from concurrent.futures import ThreadPoolExecutor
from inv_config import PORTS, TIMEOUT, CONC

socket.setdefaulttimeout(TIMEOUT)

def check_port(ip: str, port: int):
    try:
        with socket.create_connection((ip, port), timeout=TIMEOUT) as s:
            s.settimeout(TIMEOUT)
            # opportunistic banner read
            try:
                data = s.recv(128)
                banner = data.decode('utf-8','ignore').strip()
            except Exception:
                banner = None
            return port, banner
    except Exception:
        return None

def sample_ports(ip: str):
    open_ports = []
    banners = {}
    with ThreadPoolExecutor(max_workers=min(CONC, len(PORTS))) as ex:
        for res in ex.map(lambda p: check_port(ip,p), PORTS):
            if not res: continue
            p,b = res
            open_ports.append(p)
            if b: banners[str(p)] = b
    return {"ports": sorted(open_ports), "banners": banners}
```

---

## 6) Optional: Nmap integration
**What**: Run Nmap for accurate service detection and OS guesses.  
**Why**: Deeper fingerprinting when authorized.
```python
# file: inv_nmap.py
import subprocess, json

def nmap_quick(ip: str) -> dict:
    # requires `nmap` installed on the host
    cmd = ["nmap","-Pn","-T4","-sS","-sV","-oX","-", ip]
    xml = subprocess.check_output(cmd, text=True)
    # parse via python-nmap or xsltproc→json; placeholder returns raw xml
    return {"nmap_xml": xml}
```
Limit scope and speed for shared networks. Prefer `-Pn` to skip extra ICMP probes if blocked.

---

## 7) Optional: SNMP device info
**What**: Read SNMP v2c/v3 basics like `sysName` and `sysDescr`.  
**Why**: Network gear and printers expose rich identity data.
```python
# file: inv_snmp.py
from pysnmp.hlapi import *

def snmp_basic(ip: str, community: str="public") -> dict:
    oids = {
      'sysDescr': '1.3.6.1.2.1.1.1.0',
      'sysName':  '1.3.6.1.2.1.1.5.0'
    }
    out = {}
    for k, oid in oids.items():
        it = getCmd(SnmpEngine(), CommunityData(community, mpModel=1), UdpTransportTarget((ip,161), timeout=0.5, retries=1), ContextEngineId(), ObjectType(ObjectIdentity(oid)))
        errInd, errStat, errIdx, varBinds = next(it)
        if not errInd and not errStat:
            out[k] = str(varBinds[0][1])
    return {"snmp": out} if out else {}
```
Use SNMP **v3** for security in production; v2c examples are for lab use.

---

## 8) Export reports (CSV/JSON)
**What**: Persist results for sharing and diffing.  
**Why**: Track changes over time and integrate with other tools.
```python
# file: inv_export.py
from __future__ import annotations
from pathlib import Path
import csv, json

FIELDS = ["ip","alive","rtt_ms","hostname","reverse_dns","mac","vendor","ports","banners","snmp"]

def write_json(recs: list[dict], path: Path):
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(recs, indent=2), encoding='utf-8')

def write_csv(recs: list[dict], path: Path):
    path.parent.mkdir(parents=True, exist_ok=True)
    with path.open('w', newline='', encoding='utf-8') as f:
        w = csv.DictWriter(f, fieldnames=FIELDS)
        w.writeheader()
        for r in recs:
            row = {k: r.get(k) for k in FIELDS}
            # flatten for CSV
            row['ports'] = ",".join(str(p) for p in r.get('ports',[]) or [])
            row['banners'] = json.dumps(r.get('banners',{}), ensure_ascii=False)
            row['snmp'] = json.dumps(r.get('snmp',{}), ensure_ascii=False)
            w.writerow(row)
```

---

## 9) Putting it together: inventory CLI
**What**: End‑to‑end script that sweeps, enriches, and exports.
```python
# file: inventory.py
from __future__ import annotations
from concurrent.futures import ThreadPoolExecutor
from tqdm import tqdm
from inv_config import NET
from inv_ping import ping_sweep
from inv_dns import reverse_lookup
from inv_arp import arp_mac
from inv_ports import sample_ports
from inv_export import write_json, write_csv
# optional imports
try:
    from inv_snmp import snmp_basic
except Exception:
    snmp_basic = lambda ip: {}

IPs = [str(ip) for ip in NET]
base = ping_sweep(IPs)

records = []
with ThreadPoolExecutor(max_workers=64) as ex:
    for ip in tqdm(IPs, desc='enrich'):
        rec = {"ip": ip, **base.get(ip, {})}
        rec.update(reverse_lookup(ip))
        rec.update(arp_mac(ip))
        rec.update(sample_ports(ip))
        # uncomment for SNMP where allowed
        # rec.update(snmp_basic(ip, community='public'))
        records.append(rec)

write_json(records, Path('out/inventory.json'))
write_csv(records,  Path('out/inventory.csv'))
print('wrote out/inventory.{json,csv}')
```
Run:
```sh
python inventory.py
```
Expected output:
```
wrote out/inventory.{json,csv}
```
Open `out/inventory.csv` in Excel or Numbers.

---

## 10) Scheduling and deltas
**What**: Run nightly and highlight changes.  
**Why**: Detect new/removed hosts and service drift.
- Schedule with `026_TASK_SCHEDULING.md` (launchd/cron/APScheduler).  
- Write each run to `out/history/YYYY-MM-DD.json`.  
- Diff two runs on `ip`, `ports`, and `vendor`; email summary via `041_EMAIL_REPORTING.md`.

Delta skeleton:
```python
import json
prev = json.load(open('out/history/2025-10-05.json'))
curr = json.load(open('out/history/2025-10-06.json'))
prev_index = {r['ip']: r for r in prev}
changes = []
for r in curr:
    p = prev_index.get(r['ip'])
    if not p:
        changes.append({'ip': r['ip'], 'change': 'NEW'})
    elif set(r.get('ports',[])) != set(p.get('ports',[])):
        changes.append({'ip': r['ip'], 'change': 'PORTS', 'from': p.get('ports',[]), 'to': r.get('ports',[])})
print(changes)
```

---

## 11) Troubleshooting
- **ICMP blocked**: rely on TCP sampling; treat ICMP negative as unknown.  
- **ARP empty**: ping first or run with admin rights; ARP is per‑interface and volatile.  
- **Reverse DNS slow**: set a short resolver lifetime; skip on timeouts.  
- **Banner gibberish**: some services speak binary; keep reads short and tolerant.  
- **Legal/Policy**: get approval for port scans and SNMP use. Use `-Pn` in Nmap when ICMP is filtered.  
- **macOS sandbox prompts**: grant Terminal Full Disk Access only if needed for output paths.

---

## 12) Recap
```plaintext
Load config → ICMP sweep → DNS/ARP enrich → sample TCP ports + banners → optional Nmap/SNMP → export CSV/JSON → schedule + diff
```

**Next**: Visualize service counts in `011_GRAPHS.md`, notify changes with `031_NOTIFICATION_AUTOMATION.md`, and package as a CLI with `035_SCRIPT_PACKAGING.md`. 