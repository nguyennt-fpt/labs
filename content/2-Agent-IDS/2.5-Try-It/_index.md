---
title : "Try It"
date: "2026-11-03"
weight : 5
chapter : false
pre : " <b> 2.5. </b> "
---

### 1. Send a few attacks

Get your DVWA session cookie from the browser (DevTools → Application → Cookies → `PHPSESSID`), then from **your own
machine** run:

```bash
# SQL injection — write spaces as %20, not '+'
curl "http://<EC2_IP>/vulnerabilities/sqli/?id=1'%20OR%201=1--&Submit=Submit" \
  -b "security=low; PHPSESSID=<PHPSESSID>"

# Reflected XSS
curl "http://<EC2_IP>/vulnerabilities/xss_r/?name=%3Cscript%3Ealert%281%29%3C%2Fscript%3E" \
  -b "security=low; PHPSESSID=<PHPSESSID>"

# Command injection
curl "http://<EC2_IP>/vulnerabilities/exec/" --data "ip=127.0.0.1;id&Submit=Submit" \
  -b "security=low; PHPSESSID=<PHPSESSID>"

# Path traversal / LFI
curl "http://<EC2_IP>/vulnerabilities/fi/?page=../../../../../../etc/passwd" \
  -b "security=low; PHPSESSID=<PHPSESSID>"
```

And one ordinary request, to see the agent leave it alone:

```bash
curl "http://<EC2_IP>/vulnerabilities/sqli/?id=1&Submit=Submit" -b "security=low; PHPSESSID=<PHPSESSID>"
```

{{% notice warning %}}
With the default settings (`BLOCK_ENABLED=true`) the agent **really blocks the IP it judges malicious — that is your IP**.
Port 80 will time out for an hour. SSH keeps working because only web traffic to the containers goes through the
blocking rule. To unblock yourself:

`docker exec agent python -c "from tools.firewall import unblock_ip; unblock_ip('<YOUR_IP>')"`

To watch decisions without blocking, start the agent with
`BLOCK_ENABLED=false docker compose up -d --force-recreate --no-deps agent`.
{{% /notice %}}

{{% notice tip %}}
Testing **on the instance itself**? Use `127.0.0.1`, not `localhost`. curl prefers IPv6 (`::1`) for `localhost`, and
Suricata's address variables are IPv4-only, so you would see no alerts at all.
{{% /notice %}}

### 2. Watch the decisions

```bash
docker logs -f agent                                       # one line per decision, with the reason
docker exec suricata tail -f /var/log/suricata/fast.log    # raw IDS alerts
```

Open the dashboard at `http://<EC2_IP>:5000`. For each event you see the action, attack type, confidence, MITRE
ATT&CK technique, severity, the model's reason and an analyst note. A reason that does not match the evidence (for
example it mentions an IDS alert that never fired) is marked **⚠ unverified**.

### 3. See who is blocked

```bash
docker exec agent python -c \
  "from tools.firewall import get_blocked_ips; import json; print(json.dumps(get_blocked_ips(), indent=2))"
docker exec agent ipset list agent-block        # the kernel set, with the seconds left per IP
```

Blocks expire by themselves after one hour, even if the agent restarts.

### 4. Approve a block yourself (human in the loop)

By default high-risk blocks are auto-approved so the benchmark can run unattended. To make them wait for you:

```bash
AUTO_APPROVE=false HITL_ALL_BLOCKS=true docker compose up -d --force-recreate --no-deps agent
```

Send an attack again. The decision now appears on the dashboard as **pending**, with three buttons (it asks for the
`APPROVAL_TOKEN` you set during deployment):

| Button | Result |
|---|---|
| Approve | block the IP |
| Warn | record it as an attack but do not firewall |
| Benign | false positive → `log` (and, with memory on, a lesson for the agent) |

Prefer the terminal?

```bash
docker exec agent python approve.py                  # list pending items
docker exec agent python approve.py <eid> approve    # or warn / benign
```

Nobody answers within `HITL_TIMEOUT_S` (15 minutes by default)? The item becomes `warn` — **an unreviewed block never
happens**.

There is also a small interactive demo of the same gate:

```bash
docker exec -e AUTO_APPROVE=false -it agent python demo_hitl.py
```
