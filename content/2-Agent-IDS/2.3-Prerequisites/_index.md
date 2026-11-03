---
title : "Prerequisites"
date: "2026-11-03"
weight : 3
chapter : false
pre : " <b> 2.3. </b> "
---

### 1. An AWS GPU instance

The model needs a GPU with about 10 GB of memory. The project uses:

| Setting | Value |
|---|---|
| Instance type | `g4dn.xlarge` (NVIDIA Tesla T4 16 GB, 4 vCPU, 16 GB RAM) |
| AMI | Ubuntu Server 24.04 LTS (or the *Deep Learning Base OSS Nvidia Driver GPU AMI*, which has the driver preinstalled) |
| Storage | 50 GB gp3 (the model alone is ~9 GB) |
| Region | any region with g4dn capacity (the project used `us-east-1`) |

{{% notice note %}}
New AWS accounts often have a **vCPU quota of 0 for G instances**. Request an increase for
"Running On-Demand G and VT instances" (at least 4 vCPUs) in Service Quotas before you start; approval can take a while.
{{% /notice %}}

**Cost:** an on-demand g4dn.xlarge is roughly 0.5 USD per hour in us-east-1. **Stop the instance whenever you are not
using it** — a stopped instance only costs the disk.

### 2. Security group

Open only what you need, and **only to your own IP**:

```
Inbound rules:
  SSH    22    <YOUR_IP>/32
  HTTP   80    <YOUR_IP>/32     # DVWA through Nginx
  Custom 5000  <YOUR_IP>/32     # dashboard

Outbound: all traffic (to pull images, rules and the model)
```

{{% notice warning %}}
Never open port 80 to `0.0.0.0/0`. DVWA is vulnerable on purpose; anyone who reaches it can run commands on the box.
{{% /notice %}}

Home internet IPs change. If SSH suddenly times out, check that your current IP (`curl https://checkip.amazonaws.com`)
is still the one in the security group.

### 3. An SSH key pair

Create or choose a `.pem` key pair when launching the instance and keep it **outside any git repository** (or make sure
`*.pem` is in `.gitignore`).

On Windows, the built-in OpenSSH may refuse the key with "UNPROTECTED PRIVATE KEY FILE". The `ssh` that ships with
**Git for Windows** (Git Bash) does not have this problem.

### 4. Tools on your own machine

| Tool | Used for |
|---|---|
| Git | cloning the project |
| SSH client (Git Bash on Windows) | connecting to the instance |
| AWS CLI (optional) | starting/stopping the instance from the terminal |
| Python 3.10+ (optional) | running the offline tests: `python -m pytest tests -q` |

### 5. Source code

```bash
git clone https://github.com/nguyennt-fpt/agent-ids.git
```

| Folder | Content |
|---|---|
| `agent/` | the LangGraph agent, prompts, tools (alert reader, firewall, memory) |
| `dashboard/` | Flask dashboard |
| `ids/` | Suricata config, custom rules, rule-update script |
| `webserver/` | Nginx config |
| `benchmark/` | corpus, replay, scoring, statistics, prompt-injection and learning-curve experiments |
| `tests/` | offline pytest suite (no Docker or GPU needed) |
| `ops/ec2.sh` | start/stop helper for the lab instance |
