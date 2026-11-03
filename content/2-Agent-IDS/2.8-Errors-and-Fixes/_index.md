---
title : "Errors and Fixes"
date: "2026-11-03"
weight : 8
chapter : false
pre : " <b> 2.8. </b> "
---

Every problem below actually happened while building and measuring this project. Each one is written as
**symptom → cause → fix**, so you can search this page for the error you are looking at.

### A. Infrastructure and deployment

**1. Suricata sees no traffic at all**
- *Symptom:* no alerts, even for an obvious `OR 1=1`.
- *Cause:* the config listened on `eth0`, but EC2 Nitro instances name the interface `ens5`; local test traffic also goes
  over `lo`.
- *Fix:* set the right interface in `ids/suricata.yaml` (check with `ip -br link`), and keep `HTTP_SERVERS: any`.

**2. `curl localhost` raises no alert**
- *Cause:* curl resolves `localhost` to IPv6 `::1`; Suricata's address variables are IPv4-only.
- *Fix:* use `127.0.0.1` or `curl -4`.

**3. SQLi with `+` for spaces is not detected**
- *Cause:* the custom rule's regular expression needs real whitespace after URL decoding; `+` is only a space in forms.
- *Fix:* write spaces as `%20` when testing (a real rule weakness — an attacker could use it).

**4. suricata-update "succeeds" but no ET Open rules load**
- *Cause:* 16 Modbus/DNP3 rules cannot load in this lab, so suricata-update's self-test fails and it silently restores
  the previous (empty) rule file.
- *Fix:* `--disable-conf ids/rules/disable.conf`, and read the output for `Suricata test failed`. `ids/update_rules.sh`
  now stops with an error in that case.

**5. Every DVWA page triggers an alert, even benign ones**
- *Cause:* ET rule `2221036` (HTTP response excessive header repetition) fires on DVWA's own responses.
- *Fix:* disabled in `disable.conf` — the same tuning for every mode, and an appendix re-run on untuned rules.

**6. Scripts fail on the server with `$'\r': command not found`**
- *Cause:* `git archive` on Windows with `core.autocrlf=true` exports CRLF line endings.
- *Fix:* `git -c core.autocrlf=false archive ...`.

**7. Re-deploying with `tar` fails with "Permission denied" in `ids/`**
- *Cause:* the Suricata container chowns files in its bind-mounted folder to its own user on every start.
- *Fix:* `sudo chown -R ubuntu:ubuntu ~/agent-ids` before extracting.

**8. The agent hangs forever reading the Nginx log**
- *Cause:* the official Nginx image links `access.log` to `/dev/stdout`; opening it blocks.
- *Fix:* write the JSON log to a real file (`access.json.log`) and check `os.path.isfile` before reading.

**9. Ollama container healthcheck always fails**
- *Cause:* the image has no `curl`.
- *Fix:* use `ollama list` as the healthcheck command.

**10. Code changes do not take effect**
- *Cause:* the agent and dashboard code is baked into their images.
- *Fix:* `docker compose up -d --build agent` (and `docker compose build dashboard`). Config files that are bind-mounted
  (Nginx, Suricata) only need a restart.

### B. GPU and the host

**11. Everything becomes very slow; `ollama ps` shows `100% CPU`**
- *Symptom:* `docker exec ollama nvidia-smi` prints `Failed to initialize NVML: Unknown Error`.
- *Cause:* a known issue of the NVIDIA container toolkit with Docker's systemd cgroup driver: a `systemctl daemon-reload`
  (for example after a package upgrade) can cut the container off from the GPU.
- *Fix:* `docker compose restart ollama`. **Check `docker exec ollama nvidia-smi -L` before every measurement.**

**12. A benchmark run times out with 52 undecided requests**
- *Cause:* Ubuntu's `apt-daily` timer ran during the benchmark. apt downloads over plain HTTP, Suricata alerted 211 times
  on apt's user agent, and the agent spent ~12 minutes of GPU time triaging junk alerts while real requests queued.
- *Fix:* the noisy rule disabled; apt timers turned off (update by hand at session start); and duplicate alerts (same
  IP + rule within 5 minutes) are now judged only once. *Lesson:* an alert flood is also a way to slow the agent down.

**13. After an automatic upgrade the GPU disappears completely**
- *Symptom:* `Driver/library version mismatch` on the host, Ollama cannot start.
- *Cause:* unattended-upgrades installed new NVIDIA libraries while the old driver was still loaded.
- *Fix:* let dpkg finish, then `sudo reboot`. Prevent it by turning off the apt timers (see 12).

### C. Measurement bugs (the dangerous kind — wrong numbers, no error)

**14. Runs 2–5 of every mode scored recall 0**
- *Cause:* `docker compose up -d` does not recreate a container whose config did not change, so the agent's memory of
  already-decided request ids survived between runs and it skipped everything.
- *Fix:* `--force-recreate` on every run.

**15. The agent re-processed the previous run's requests**
- *Cause:* at startup it read the last 100 lines of the Nginx log.
- *Fix:* start reading from the end of every log file, and track a byte offset afterwards.

**16. Scoring started before the agent had finished**
- *Cause:* "no new decision for 15 seconds" was taken as "done", but one LLM call can take longer.
- *Fix:* wait until **every** sent request has a decision (with a long safety timeout), and report `undecided` separately.

**17. A truncated LLM answer marked a whole batch as benign (fail-open)**
- *Fix:* salvage every complete item from broken JSON, treat unknown actions as `warn`, give IDS events without a decision
  the rule-based verdict, retry web events twice, then `warn`. The system now fails **closed**.

**18. Benign `id=1` was flagged as SQLi**
- *Cause:* two artifacts of the benchmark itself: the random hex `eid` in the URL looked like an encoded payload, and in
  a batch the request was judged next to real SQLi on the same page.
- *Fix:* hide the `eid` from the model, and judge **one event per LLM call** (`TRIAGE_BATCH=1`).

**19. IDS alerts contained other requests' payloads**
- *Cause:* the replay reused one keep-alive HTTP connection for every simulated client, so one packet carried several
  requests.
- *Fix:* one connection per request.

**20. False alarms grew from run to run**
- *Cause:* the investigate tools saw the same IP sending the same request in earlier runs and called it "repeated".
- *Fix:* tools only look at the current run (since agent start, 15-minute window).

**21. A calibration score that meant nothing**
- *Cause:* the scoring script read `confidence` as "how sure I am of my decision", the model used it as "how likely this
  is an attack". Nobody had defined it.
- *Fix:* define `confidence` = probability of attack in the schema and the prompt. *Lesson:* before an expensive run, check
  what **every** metric means, not just the headline ones.

**22. The model wrote "IDS severity 1" when no IDS alert existed**
- *Cause:* the phrase was copied from an example in the prompt.
- *Fix:* removed from the examples, and a grounding check marks such reasons **unverified** and sends those blocks to a
  human.

### D. Working from Windows

**23. `ssh` refuses the key: UNPROTECTED PRIVATE KEY FILE** — use Git Bash's `ssh` instead of the Windows one.

**24. Remote commands lose their quotes in PowerShell** — PowerShell 5.1 strips nested `"` when calling `ssh.exe`; use
single quotes inside the remote command, or Git Bash.

**25. A remote heredoc script stops silently halfway** — a command inside `ssh host 'bash -s' <<'EOF'` that reads stdin
(`docker compose`, scripts) swallows the rest of the script. Add `< /dev/null` to those commands.

**26. SSH times out one day** — your home IP changed; update the security group.
