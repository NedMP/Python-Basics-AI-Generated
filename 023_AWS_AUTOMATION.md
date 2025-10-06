

# AWS Automation with Python (boto3) — updated Oct 6, 2025

Practical guide to automating AWS from Python. Covers credentials and regions, creating a boto3 `Session`, common services (EC2, S3, Lambda, IAM), pagination and tagging, STS role assumption, retries/timeouts, and infrastructure deployment with CloudFormation/CDK interop. Use with `002_SETUP.md` for venv and `.env` handling.

---

**Assumptions**  
- macOS + zsh, Python 3.12–3.13 in a venv.  
- AWS CLI installed and you have permissions in at least one account.  
- Least privilege and tagging are enforced in your org.

**Install**
```sh
pip install boto3 botocore "botocore[crt]" python-dotenv rich
# optional helpers
pip install mypy-boto3-stubs
```

---

## Table of Contents
- [0) Credentials and regions](#0-credentials-and-regions)
- [1) Sessions, clients, and resources](#1-sessions-clients-and-resources)
- [2) EC2: list, start, stop](#2-ec2-list-start-stop)
- [3) S3: upload, download, list](#3-s3-upload-download-list)
- [4) Lambda: invoke and deploy](#4-lambda-invoke-and-deploy)
- [5) IAM: users/roles and policies](#5-iam-usersroles-and-policies)
- [6) Pagination, filters, and tags](#6-pagination-filters-and-tags)
- [7) STS: assume role (multi‑account)](#7-sts-assume-role-multi-account)
- [8) Retries, timeouts, and endpoints](#8-retries-timeouts-and-endpoints)
- [9) CloudFormation/CDK interop](#9-cloudformationcdk-interop)
- [10) Troubleshooting](#10-troubleshooting)
- [11) Recap](#11-recap)

---

## 0) Credentials and regions
**What**: boto3 pulls credentials from the **default provider chain**: env vars → shared config/credentials files → SSO/cache → IAM role (EC2/ECS/Lambda).  
**Why**: One code path works across local, CI, and AWS runtimes.

Set up AWS CLI once:
```sh
brew install awscli
aws configure sso     # or: aws configure
```
Useful env vars (override per run):
```sh
export AWS_PROFILE=default
export AWS_REGION=ap-southeast-2     # Sydney
```
Files: `~/.aws/config`, `~/.aws/credentials`.

---

## 1) Sessions, clients, and resources
**What**: A `Session` encapsulates profile/region. Use **clients** for full API coverage, **resources** for higher‑level object wrappers (where available).
```python
import boto3, os

session = boto3.Session(profile_name=os.getenv("AWS_PROFILE"), region_name=os.getenv("AWS_REGION", "ap-southeast-2"))
ec2 = session.client("ec2")
s3  = session.resource("s3")
```
Tip: Prefer passing a `Session` explicitly rather than relying on globals when writing libraries.

---

## 2) EC2: list, start, stop
**What**: Query instance state and control power.
```python
import boto3
session = boto3.Session(region_name="ap-southeast-2")
ec2 = session.client("ec2")

# list running instances with Name tag
paginator = ec2.get_paginator("describe_instances")
for page in paginator.paginate(Filters=[{"Name":"instance-state-name","Values":["running"]}]):
    for r in page["Reservations"]:
        for i in r["Instances"]:
            name = next((t["Value"] for t in i.get("Tags",[]) if t["Key"]=="Name"), "")
            print(i["InstanceId"], name, i["State"]["Name"]) 

# stop/start (idempotent API)
ec2.stop_instances(InstanceIds=["i-0123456789abcdef0"])    # no error if already stopped
ec2.start_instances(InstanceIds=["i-0123456789abcdef0"])   # asynchronously starts
```
For waiters:
```python
waiter = ec2.get_waiter("instance_stopped")
waiter.wait(InstanceIds=["i-0123456789abcdef0"])  # blocks until state reached
```

---

## 3) S3: upload, download, list
**What**: Store and retrieve objects. Use the high‑level **resource** or **client** with `upload_file`/`download_file` for retries and multipart.
```python
import boto3
s3 = boto3.resource("s3")

bucket = s3.Bucket("my-bucket")
# upload with sensible defaults
bucket.upload_file("report.csv", "reports/2025-10-06/report.csv")

# download
bucket.download_file("reports/2025-10-06/report.csv", "report_copy.csv")

# list
for obj in bucket.objects.filter(Prefix="reports/"):
    print(obj.key, obj.size)
```
Public access: keep buckets private; use pre‑signed URLs instead:
```python
import boto3, datetime
s3c = boto3.client("s3")
url = s3c.generate_presigned_url(
    "get_object",
    Params={"Bucket":"my-bucket","Key":"reports/2025-10-06/report.csv"},
    ExpiresIn=3600,
)
print(url)
```

---

## 4) Lambda: invoke and deploy
**What**: Run code on demand; ship zips for simple deployments.
```python
import boto3, json
lam = boto3.client("lambda")

# invoke
resp = lam.invoke(FunctionName="hello-fn", Payload=json.dumps({"name":"Ned"}).encode("utf-8"))
print(json.loads(resp["Payload"].read()))
```
Deploy basic zip:
```python
import boto3
lam = boto3.client("lambda")
with open("package.zip","rb") as f:
    lam.update_function_code(FunctionName="hello-fn", ZipFile=f.read(), Publish=True)
```
Tip: For larger, repeatable deploys prefer **SAM** or **CDK**.

---

## 5) IAM: users/roles and policies
**What**: Manage identities and permissions. Use caution and least privilege.
```python
import boto3, json
iam = boto3.client("iam")

# create a policy (example: read-only S3 folder)
policy_doc = {
  "Version": "2012-10-17",
  "Statement": [{
    "Effect":"Allow",
    "Action":["s3:GetObject","s3:ListBucket"],
    "Resource":[
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/reports/*"
    ]
  }]
}
res = iam.create_policy(PolicyName="ReadonlyReports", PolicyDocument=json.dumps(policy_doc))
print(res["Policy"]["Arn"])  # attach to role/user as needed
```
Attach to a role:
```python
iam.attach_role_policy(RoleName="report-reader", PolicyArn=res["Policy"]["Arn"])  # requires role to exist
```
Notes:
- Prefer **roles** over long‑lived users. Rotate and scope secrets.

---

## 6) Pagination, filters, and tags
**What**: Many AWS APIs are paginated; use paginators. Enforce tags for ownership/cost.
```python
import boto3
session = boto3.Session(region_name="ap-southeast-2")
ec2 = session.client("ec2")

p = ec2.get_paginator("describe_volumes")
for page in p.paginate(Filters=[{"Name":"tag:owner","Values":["ned"]}]):
    for v in page["Volumes"]:
        print(v["VolumeId"], v.get("Tags", []))
```
Set tags on create or after:
```python
ec2.create_tags(Resources=["i-0123456789abcdef0"], Tags=[{"Key":"owner","Value":"ned"},{"Key":"env","Value":"dev"}])
```

---

## 7) STS: assume role (multi‑account)
**What**: Jump from a source account/profile to a target account role.
```python
import boto3
sts = boto3.client("sts")
resp = sts.assume_role(
    RoleArn="arn:aws:iam::123456789012:role/automation",
    RoleSessionName="ned-session",
    DurationSeconds=3600,
)
creds = resp["Credentials"]
assumed = boto3.Session(
    aws_access_key_id=creds["AccessKeyId"],
    aws_secret_access_key=creds["SecretAccessKey"],
    aws_session_token=creds["SessionToken"],
    region_name="ap-southeast-2",
)
print(assumed.client("sts").get_caller_identity())
```
Prefer SSO and profile‑chained roles in `~/.aws/config` for day‑to‑day use.

---

## 8) Retries, timeouts, and endpoints
**What**: Make calls resilient and fast.
```python
import boto3
from botocore.config import Config

cfg = Config(
  retries={"max_attempts": 10, "mode": "standard"},
  connect_timeout=5, read_timeout=60,
)
s3 = boto3.client("s3", config=cfg, region_name="ap-southeast-2")
```
Use VPC endpoints for private access where required. Keep regions explicit for clarity.

---

## 9) CloudFormation/CDK interop
**What**: Manage stacks declaratively; drive deployments from Python.
```python
import boto3, json
cf = boto3.client("cloudformation")
with open("template.json","r") as f:
    template_body = f.read()

res = cf.create_stack(
  StackName="demo-stack",
  TemplateBody=template_body,
  Capabilities=["CAPABILITY_NAMED_IAM"],
  Parameters=[{"ParameterKey":"Env","ParameterValue":"dev"}],
)
print(res["StackId"])  # use describe_stack_events for progress
```
For CDK, call `subprocess.run(["cdk","deploy"])` from Python if you need to orchestrate builds.

---

## 10) Troubleshooting
- **Auth errors (`AccessDenied`, `UnrecognizedClientException`)**: wrong profile/region or missing perms. Run `aws sts get-caller-identity` to verify caller.  
- **Endpoint connection errors**: region mismatch; check `AWS_REGION` and explicitly set in `Session`.  
- **ThrottlingException**: add retries/backoff and reduce concurrency.  
- **S3 403 vs 404**: 403 often means the object exists but you lack permission.  
- **Lambda size limits**: zip under limit or use container image.  
- **IAM policy wildcards**: avoid `*`; scope to actions and ARNs.  
- **SSL/cert issues on macOS**: update `certifi` and ensure system trust store is current.

---

## 11) Recap
```plaintext
Configure AWS CLI/SSO → create Session → use clients/resources → automate EC2/S3/Lambda/IAM → paginate and tag → assume roles for multi‑account → add retries/timeouts → deploy with CloudFormation/CDK when needed
```

**Next**: Schedule jobs with `026_TASK_SCHEDULING.md`, notify results via `031_NOTIFICATION_AUTOMATION.md`, and feed data into reports from `013_CSVEXCEL.md` or emails via `014_EMAILS.md`. 