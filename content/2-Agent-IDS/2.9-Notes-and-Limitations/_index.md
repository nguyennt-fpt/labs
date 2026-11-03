---
title : "Notes and Limitations"
date: "2026-11-03"
weight : 9
chapter : false
pre : " <b> 2.9. </b> "
---

### 1. Security notes

- **Isolate the lab.** DVWA is vulnerable on purpose. Restrict ports 22, 80 and 5000 to your own IP.
- **Secrets stay out of git:** the SSH key (`*.pem`) and `.env` (`APPROVAL_TOKEN`) are gitignored. Never print the token
  in logs or screenshots.
- **The dashboard can change exactly one thing** — the answer to a pending block — and only with the token. Any new
  write endpoint must be protected the same way.
- **Tools are read-only.** The model can look at logs but cannot block anything; only `actions.py` enforces decisions,
  after the risk gate.
- **Blocks expire.** IPs are added to an `ipset` with a one-hour timeout, removed by the kernel even if the agent crashes.
- **Spoofed IPs:** Suricata trusts `X-Forwarded-For` from every client (the replay needs it). An alert that cannot be joined
  with an Nginx line still carries the client-supplied IP, so the agent could block the wrong address. In production,
  trust that header only from your own proxy.

### 2. What the numbers do not prove

- **Small scope:** 147 test requests on one application. Real traffic contains far fewer attacks, so precision would
  be lower in practice.
- **Payloads never execute:** the replay does not log in to DVWA, so vulnerable pages redirect to the login page. Only
  detection on the request is measured; IDS rules that look at the response never get a chance to fire.
- **The test set was used three times** (three final runs as the prompt improved). v4 was tuned only on dev, but having
  seen earlier test results can still make the numbers slightly optimistic. Held_out is cleaner but contains only
  attacks, and is too easy (both prompt versions score 61/61).
- **The prompt-injection set is too easy:** with 0% bypass even without defenses, it cannot tell which defense helps.
- **Throughput was not measured:** the benchmark checks decision quality at one request every 6 seconds.

### 3. Would this work in production?

Not as an inline filter on every request. One T4 handles about **15 requests per minute** sequentially; a real website
can receive hundreds per second. The realistic design is layered:

1. A **WAF in the request path** blocks known attacks instantly and cheaply.
2. Only **suspicious events** (IDS alerts, WAF anomaly scores, unusual patterns) go to the **LLM agent**, which triages,
   investigates and writes a note for the analyst.
3. **Risky actions wait for a human.**

The agent's ~15-second reaction time is fine for that role — a human SOC analyst takes minutes to hours — and it still
stops multi-step attacks (scan → probe → exploit). Serving the model with a batching server such as vLLM, a bigger GPU,
or a small model for first-pass filtering would raise throughput.

### 4. Lessons learned

- **Measure at the decision layer with a unique id per request.** It makes every number traceable to a request.
- **Write down the success criterion before you look at the data**, and keep a test set you do not tune on.
- **Report negative results.** Memory did not help; the injection set is too easy — both are useful findings.
- **Most "model problems" were measurement problems.** Batching order, a hex id in the URL and a copied example phrase
  caused more false alarms than the model's reasoning did.
- **The cheapest win was the data, not the model:** adding the web log beat any prompt change.
