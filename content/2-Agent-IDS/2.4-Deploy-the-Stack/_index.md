---
title : "Deploy the Stack"
date: "2026-11-03"
weight : 4
chapter : false
pre : " <b> 2.4. </b> "
---

All commands below run **on the EC2 instance** unless stated otherwise. Connect first:

```bash
ssh -i your-key.pem ubuntu@<EC2_PUBLIC_DNS>
```

### Step 1 — Install Docker (from Docker's own repository)

Ubuntu's archive does not ship `docker-compose-plugin`, so use Docker's official repository:

```bash
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update -y
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER && newgrp docker
```

### Step 2 — NVIDIA driver and container toolkit

Skip the driver part if you used the Deep Learning AMI.

```bash
# driver (plain Ubuntu AMI only), then reboot and SSH back in
sudo apt-get install -y ubuntu-drivers-common
sudo ubuntu-drivers autoinstall
sudo reboot

# NVIDIA Container Toolkit
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list
sudo apt-get update -y && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

# check: you should see "Tesla T4"
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi
```

### Step 3 — Turn off automatic updates (recommended for a lab)

Ubuntu's `apt-daily` timers download and install packages in the background. In this project that **broke two
measurement runs** (see [Errors and fixes](../2.8-errors-and-fixes/)). Turn them off and update by hand instead:

```bash
sudo systemctl disable --now apt-daily.timer apt-daily-upgrade.timer
# at the start of each work session:
sudo apt update && sudo apt upgrade -y
# reboot if NVIDIA, systemd or kernel packages were upgraded
```

### Step 4 — Get the code onto the instance

```bash
git clone https://github.com/nguyennt-fpt/agent-ids.git ~/agent-ids
cd ~/agent-ids
```

{{% notice tip %}}
Copying from a **Windows** machine instead? Use `git -c core.autocrlf=false archive --format=tar HEAD | ssh ... 'tar -xf - -C ~/agent-ids'`.
A plain copy can turn every file into CRLF line endings and the shell scripts stop working.
{{% /notice %}}

### Step 5 — Download the Suricata rules (ET Open)

```bash
docker run --rm -v $(pwd)/ids/rules:/etc/suricata/rules jasonish/suricata:latest \
  suricata-update --no-reload --disable-conf /etc/suricata/rules/disable.conf \
  --output /etc/suricata/rules
```

`disable.conf` turns off rules that cannot load in this lab (Modbus/DNP3) and a few noisy ones. Without it,
suricata-update's self-test fails and it **silently keeps an empty rule file**. Once the stack is running you can use
`bash ids/update_rules.sh`, which also verifies the result and restarts Suricata.

### Step 6 — Set the dashboard approval token (optional)

The dashboard is read-only, except for one action: answering blocks that wait for a human. That action needs a token:

```bash
echo "APPROVAL_TOKEN=$(openssl rand -hex 16)" > .env    # .env is gitignored
```

Without a token the approval buttons are disabled.

### Step 7 — Start everything

```bash
docker compose up -d
docker exec ollama ollama pull qwen2.5:14b          # ~9 GB, once
docker exec ollama ollama pull nomic-embed-text     # only needed for experience memory
docker compose ps
```

Expected:

```
NAME        STATUS
nginx       running   0.0.0.0:80->80/tcp
waf         running
dvwa        running
mysql       running (healthy)
suricata    running
ollama      running (healthy)   0.0.0.0:11434->11434/tcp
agent       running
dashboard   running   0.0.0.0:5000->5000/tcp
```

Check that Ollama really uses the GPU — this is the most common silent failure:

```bash
docker exec ollama nvidia-smi -L        # must print "GPU 0: Tesla T4"
```

### Step 8 — Initialise DVWA

1. Open `http://<EC2_PUBLIC_IP>/setup.php` and click **Create / Reset Database**.
2. Log in with `admin` / `password`.
3. **DVWA Security** → set the level to **Low** → Submit.

The system is now running. Continue with [Try it](../2.5-try-it/).
