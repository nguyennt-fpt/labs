---
title : "Results"
date: "2026-11-03"
weight : 7
chapter : false
pre : " <b> 2.7. </b> "
---

Final run: 2026-11-03, prompt **v4**, test split (147 requests: 76 attacks / 71 benign), 5 runs per mode. 95% confidence
intervals in brackets. Raw CSVs: [benchmark/results/](https://github.com/nguyennt-fpt/agent-ids/tree/master/benchmark/results).

### 1. Main table

| Mode | Precision | Recall | F1 | FPR | MTTB |
|---|---|---|---|---|---|
| `rule_based` (Suricata) | 1.00 | 0.32 | 0.48 | 0.00 | 2.1 s |
| `waf` (OWASP CRS) | 0.96 | 0.71 | 0.82 | 0.03 | 2.6 s |
| `agent_ids` (LLM, alerts only) | 1.00 | 0.30 | 0.46 | 0.00 | 8.3 s |
| **`agent`** (LLM, alerts + web log) | 0.91 | 0.95 | **0.93** [0.88, 0.96] | 0.10 [0.04, 0.18] | 14.7 s |
| *`agent`, previous prompt v3* | *0.85* | *0.96* | *0.90* | *0.18* | *14.7 s* |

`naive` (block every alert) gives exactly the same result as `rule_based` here.

| Difference | Recall | FPR | F1 |
|---|---|---|---|
| agent − rule_based | +0.63 [+0.52, +0.74] | +0.10 [+0.04, +0.18] | **+0.45 [+0.33, +0.58]** |
| agent − waf | +0.24 [+0.15, +0.34] | +0.08 [+0.00, +0.16] | **+0.11 [+0.04, +0.19]** |

### 2. Where the agent wins and loses

| Group (caught / total, mean of 5 runs) | n | rule_based | waf | agent |
|---|---|---|---|---|
| clear attack | 26 | 13 | 23 | 26 |
| evasion | 26 | 7 | 23 | 26 |
| recon | 24 | 4 | 8 | 20 |
| normal benign (false alarms) | 42 | 0 | 0 | 5 |
| fp_bait (false alarms) | 29 | 0 | 2 | 2 |

**On 61 new attacks (held_out):** agent 61/61, WAF 57, Suricata 35. The WAF missed 3 remote file inclusions and 1 command
injection; Suricata missed all 12 command injections. (Held_out contains only attacks, so false-alarm rates cannot be
measured on it.)

### 3. How to read this

- **The agent catches almost everything**, including encoded evasions and reconnaissance — the kinds of attacks that slip
  past signatures.
- **It still raises more false alarms** (~7 of 71 benign requests) than the WAF (2) or Suricata (0). The original
  hypothesis "the agent reduces false positives" turned out to be **wrong**: on this data the IDS had none to reduce.
- **The lead over the WAF comes from recon** (20 vs 8): a scan or password-guessing burst is only suspicious as a
  sequence, and CRS has no rule for that.
- **The data source matters more than the model.** `agent_ids` — the same LLM fed only Suricata alerts — scores like the
  plain IDS. The extra catches come from reading the web log.
- The remaining false alarms are mostly **logins with DVWA's default accounts** on the brute-force page.
- `confidence` is well calibrated (ECE 0.04); the MITRE technique is right for 85% of caught attacks.

### 4. Prompt v3 → v4

Prompt v4 makes the model write its **evidence before its verdict** and adds one worked example ("a plain page name is
normal use; `../` is traversal"). It was chosen on the dev set against a criterion fixed in advance, before any test run.
On test it removed the six `fi/?page=<name>.php` false alarms (FPR 0.18 → 0.10), at the cost of one new false alarm and
one more missed directory-scan request. The two versions' intervals still overlap, so treat the improvement as real on
those cases but not yet firmly proven.

### 5. Side experiments

| Experiment | Result |
|---|---|
| Prompt injection, 25 techniques | **0% bypass in all four conditions**, even with every defense off. In the older run (12 techniques, prompt v2) 12% got through without defenses. The current set is too easy to show which defense matters. |
| Experience memory, 3 epochs | **No measurable gain.** Memory on: recall 0.95 → 0.92, FPR flat at 0.11. Memory off: recall 0.95, FPR 0.11 → 0.13. Differences of 1–2 requests. |
| Untuned ET Open | Same conclusions. |

Negative results are reported on purpose: an experiment that can only say "yes" is not an experiment.
