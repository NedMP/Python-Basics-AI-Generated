

# Azure Automation with Python — updated Oct 6, 2025

Practical guide to managing Azure resources with Python. Covers authenticating with **Azure Identity**, using Azure SDKs to list/create/update resources, working with Storage and Key Vault, starting/stopping VMs, tag hygiene, and deploying Bicep/ARM templates. Includes CLI interop and troubleshooting. Use with `002_SETUP.md` for venv and `.env` management.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- You have an Azure tenant, subscription, and rights to read or modify resources.  
- Secrets live in `.env` or Key Vault, not in code.

**Install**
```sh
pip install azure-identity azure-mgmt-resource azure-mgmt-compute azure-storage-blob azure-keyvault-secrets
# optional
pip install msal rich
```

---

## Table of Contents
- [0) Authentication options](#0-authentication-options)
- [1) DefaultAzureCredential (recommended)](#1-defaultazurecredential-recommended)
- [2) List and create resource groups](#2-list-and-create-resource-groups)
- [3) Start/stop and query Virtual Machines](#3-startstop-and-query-virtual-machines)
- [4) Storage: upload/download blobs](#4-storage-uploaddownload-blobs)
- [5) Key Vault: read and write secrets](#5-key-vault-read-and-write-secrets)
- [6) Tags, filtering, and pagination](#6-tags-filtering-and-pagination)
- [7) Deploy Bicep/ARM templates](#7-deploy-biceparm-templates)
- [8) Azure CLI interop from Python](#8-azure-cli-interop-from-python)
- [9) Service principals and managed identity](#9-service-principals-and-managed-identity)
- [10) Troubleshooting](#10-troubleshooting)
- [11) Recap](#11-recap)

---

## 0) Authentication options
**What**: Azure Identity provides consistent auth across local dev, CI, and cloud.  
**Why**: One code path, many environments.
- **DefaultAzureCredential**: tries multiple methods in order (env vars → managed identity → shared token cache → CLI).  
- **Device Code**: interactive login in consoles.  
- **Client Secret/Cert**: daemon apps and CI.  
- **Managed Identity**: best for code running in Azure (VMs, Functions, Container Apps, etc.).

Set helpful env vars in `.env` for deterministic behavior when needed:
```
AZURE_TENANT_ID=...
AZURE_CLIENT_ID=...
AZURE_CLIENT_SECRET=...  # for client secret flow
AZURE_SUBSCRIPTION_ID=...
```
Load with `python-dotenv` if you need to force env credentials.

---

## 1) DefaultAzureCredential (recommended)
**What**: Single credential that “just works” locally and in Azure.  
**Why**: Avoids per‑environment code.
```python
# auth.py
from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import SubscriptionClient
import os

cred = DefaultAzureCredential()
sub_client = SubscriptionClient(cred)
subs = [s.subscription_id for s in sub_client.subscriptions.list()]
print("subs:", subs)

SUBSCRIPTION_ID = os.getenv("AZURE_SUBSCRIPTION_ID") or subs[0]
print("using:", SUBSCRIPTION_ID)
```
> Tip: If you use `az login`, DefaultAzureCredential will reuse that context via the shared token cache.

---

## 2) List and create resource groups
**What**: Manage containers for resources in a subscription.
```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceManagementClient
import os

sub_id = os.environ["AZURE_SUBSCRIPTION_ID"]
cred = DefaultAzureCredential()
rg = ResourceManagementClient(cred, sub_id)

# list
for g in rg.resource_groups.list():
    print(g.name, g.location)

# create/update idempotently
result = rg.resource_groups.create_or_update(
    resource_group_name="rg-demo",
    parameters={"location": "australiaeast", "tags": {"owner": "ned", "env": "dev"}}
)
print("created:", result.name)
```
Notes:
- `create_or_update` is idempotent.  
- Choose regions close to your users/compliance (e.g., `australiaeast`, `australiasoutheast`).

---

## 3) Start/stop and query Virtual Machines
**What**: Control VM lifecycle and query instance view.
```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.compute import ComputeManagementClient
import os

sub_id = os.environ["AZURE_SUBSCRIPTION_ID"]
cred = DefaultAzureCredential()
compute = ComputeManagementClient(cred, sub_id)

# list VMs in a resource group
for vm in compute.virtual_machines.list(resource_group_name="rg-demo"):
    print(vm.name)

# get power state
vm_ref = compute.virtual_machines.get("rg-demo", "vm-demo", expand='instanceView')
for s in vm_ref.instance_view.statuses:
    if s.code.startswith("PowerState"):
        print("state:", s.display_status)

# start/stop
compute.virtual_machines.begin_start("rg-demo", "vm-demo").result()
compute.virtual_machines.begin_deallocate("rg-demo", "vm-demo").result()
```
Notes:
- Use `begin_` methods for long‑running operations and call `.result()` to wait.

---

## 4) Storage: upload/download blobs
**What**: Store files in Azure Blob Storage.
```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient
import os

account_url = f"https://{os.environ['AZURE_STORAGE_ACCOUNT']}.blob.core.windows.net"
cred = DefaultAzureCredential()
svc = BlobServiceClient(account_url=account_url, credential=cred)
container = svc.get_container_client("backups")
container.create_container(exist_ok=True)

# upload
blob = container.get_blob_client("report.csv")
with open("report.csv", "rb") as f:
    blob.upload_blob(f, overwrite=True)

# download
with open("report_copy.csv", "wb") as f:
    stream = blob.download_blob()
    f.write(stream.readall())
```
> Ensure your identity has `Storage Blob Data Contributor` role on the account or container.

---

## 5) Key Vault: read and write secrets
**What**: Centralize secrets and avoid plaintext in code.
```python
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
import os

kv = os.environ["AZURE_KEY_VAULT_NAME"]  # e.g., kv-demo
url = f"https://{kv}.vault.azure.net"
cred = DefaultAzureCredential()
client = SecretClient(vault_url=url, credential=cred)

client.set_secret("db-password", "s3cr3t")
secret = client.get_secret("db-password")
print(secret.name, secret.value)
```
> Grant your principal **Secret Get/Set** permissions via Access Policies or RBAC.

---

## 6) Tags, filtering, and pagination
**What**: Enforce ownership and environment tags for cost and hygiene.
```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceGraphClient
from azure.mgmt.resourcegraph import models
import os

cred = DefaultAzureCredential()
sub = os.environ["AZURE_SUBSCRIPTION_ID"]
rgc = ResourceGraphClient(cred)
query = """
Resources
| where tags.owner == 'ned' and tags.env == 'dev'
| project name, type, location, resourceGroup
"""
req = models.QueryRequest(subscriptions=[sub], query=query)
res = rgc.resources(req)
for row in res.data:
    print(row)
```
Pagination: most `list()` methods return iterables that auto‑paginate. Convert to list only if needed.

---

## 7) Deploy Bicep/ARM templates
**What**: Declaratively create resources using templates.
```python
from azure.identity import DefaultAzureCredential
from azure.mgmt.resource import ResourceManagementClient
import json, os

sub = os.environ["AZURE_SUBSCRIPTION_ID"]
cred = DefaultAzureCredential()
rg = ResourceManagementClient(cred, sub)

# assuming you've compiled bicep → arm.json, or you have an ARM JSON template
with open("main.arm.json", "r", encoding="utf-8") as f:
    template = json.load(f)

params = {"location": {"value": "australiaeast"}, "vmName": {"value": "vm-demo"}}

poller = rg.deployments.begin_create_or_update(
    "rg-demo", "dep-demo",
    {"properties": {
        "mode": "Incremental",
        "template": template,
        "parameters": params
    }}
)
result = poller.result()
print("deployment state:", result.properties.provisioning_state)
```
> Compile Bicep: `az bicep build --file main.bicep --out main.arm.json`.

---

## 8) Azure CLI interop from Python
**What**: Reuse `az` for features not yet in SDKs or for quick wins.
```python
import subprocess, json

cmd = ["az", "vm", "list", "-g", "rg-demo", "--output", "json"]
out = subprocess.check_output(cmd, text=True)
vms = json.loads(out)
print([v["name"] for v in vms])
```
> Ensure you’ve run `az login` and selected the subscription.

---

## 9) Service principals and managed identity
**What**: Non‑interactive auth for automation.

### 9.1 Client credentials (service principal)
Create once (portal or CLI), then set env vars in CI:
```
AZURE_TENANT_ID=...
AZURE_CLIENT_ID=...
AZURE_CLIENT_SECRET=...
AZURE_SUBSCRIPTION_ID=...
```
Code is identical when using `DefaultAzureCredential()`; it will pick env creds first.

### 9.2 Managed Identity
When code runs on Azure (VM/Function/Container Apps), assign a system/user‑assigned identity and grant roles. `DefaultAzureCredential` will detect and use it automatically.

Role examples: `Reader`, `Contributor`, `Virtual Machine Contributor`, `Storage Blob Data Contributor`, `Key Vault Secrets User`.

---

## 10) Troubleshooting
- **`CredentialUnavailableError`**: no usable auth method found. Export env vars or `az login`.  
- **`AuthorizationFailed`**: missing RBAC role on the target resource. Use least privilege roles.  
- **Long‑running ops hang**: wait for `begin_*().result()` or poll status; set timeouts/retries.  
- **Storage 403**: you have control plane rights but not data plane. Grant `Storage Blob Data Contributor`.  
- **Key Vault 403**: missing secret permissions. Configure Access Policies/RBAC.  
- **Wrong subscription**: explicitly pass `AZURE_SUBSCRIPTION_ID` and verify with `az account show`.

---

## 11) Recap
```plaintext
Authenticate with DefaultAzureCredential → select subscription → list/create resource groups → manage VMs → use Blob for files → store secrets in Key Vault → tag and query with Resource Graph → deploy Bicep/ARM → use service principals or managed identity in automation
```

**Next**: Combine with `026_TASK_SCHEDULING.md` for periodic jobs, `025_MONITORING_SCRIPTS.md` for health checks, and `031_NOTIFICATION_AUTOMATION.md` for alerting. For API integrations, see `020_API_AUTOMATION.md`. 