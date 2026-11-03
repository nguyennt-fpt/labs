---
title : "Introduction"
date: "2026-11-03"
weight : 1
chapter : false
pre : " <b> 2.1. </b> "
---

### The problem

A rule-based IDS such as Suricata is fast (well under a millisecond per packet) and almost never raises a false alarm on
ordinary traffic — but it only sees what its rules describe. An attacker who **encodes a payload a little differently**, or
who **spreads an attack over many harmless-looking requests** (directory scanning, password guessing), walks past it.

Large language models are good at reading a request the way a human analyst would. The question this project asks is:

> **Does an AI agent actually detect web attacks better than a rule-based IDS — where, and at what cost?**

### What the project builds

| Layer | Component | Role |
|---|---|---|
| Target | **DVWA** (+ MySQL) | A deliberately vulnerable PHP application to attack |
| Edge | **Nginx** | Reverse proxy; writes a JSON access log of every request |
| Web firewall | **OWASP CRS on ModSecurity** | Industry-standard WAF, run in *DetectionOnly* mode as a baseline |
| Network IDS | **Suricata** | Watches a copy of the traffic; ET Open + custom rules raise alerts |
| Decision | **AI agent (LangGraph)** | Reads alerts + web log, asks the LLM, decides log / warn / block |
| Model | **Ollama + Qwen2.5-14B** | Runs locally on an NVIDIA T4 GPU — no data leaves the machine |
| Response | **ipset + iptables** | Blocks an attacker IP for one hour |
| Visibility | **Flask dashboard** | Every decision with its reason, MITRE technique and analyst note |

Everything runs with **Docker Compose** on a single **AWS EC2 g4dn.xlarge** instance.

### What it found (short version)

On a test set of 147 requests (76 attacks, 71 benign), five runs per mode:

| Mode | Recall (attacks caught) | False-positive rate | F1 |
|---|---|---|---|
| Suricata rules (`rule_based`) | 0.32 | 0.00 | 0.48 |
| OWASP CRS WAF (`waf`) | 0.71 | 0.03 | 0.82 |
| **AI agent (`agent`)** | **0.95** | 0.10 | **0.93** |

- The agent catches **almost every attack**, including encoded evasions and multi-request reconnaissance.
- It pays for that with **more false alarms** than the WAF or the IDS.
- An ablation shows the gain comes from **reading the web log** (seeing requests the IDS never alerted on), not from the
  LLM "re-thinking" IDS alerts.
- On **61 brand-new attacks** never seen during tuning, the agent caught **61/61**, the WAF 57, the IDS 35.

The full numbers, with confidence intervals and what they do *not* prove, are in [Results](../2.7-results/).

### Who this guide is for

- Students or engineers who want to **build a working IDS + LLM agent lab** themselves.
- Anyone curious about **how to measure** an AI security tool honestly instead of showing a few cherry-picked demos.

You will need some comfort with the Linux command line, Docker and basic AWS (launching an EC2 instance). No machine
learning background is required.

{{% notice warning %}}
This lab runs an **intentionally vulnerable application**. Deploy it only in an isolated environment and restrict every
open port to your own IP address.
{{% /notice %}}
