---
title : "Clean Up"
date: "2026-11-03"
weight : 10
chapter : false
pre : " <b> 2.10. </b> "
---

The GPU instance is the only expensive part. Choose how far you want to go:

### 1. Pause (keep everything for next time)

On the instance:

```bash
docker compose down          # stop the containers, keep the data and the downloaded model
```

Then **stop the instance** in the EC2 console (Instance state → Stop), or from your machine:

```bash
aws ec2 stop-instances --instance-ids <INSTANCE_ID>
```

A g4dn takes 8–10 minutes to go from `stopping` to `stopped`. While stopped you only pay for the disk. When you start it
again the public IP/DNS changes, and you should update packages by hand and check the GPU before measuring:

```bash
sudo apt update && sudo apt upgrade -y
docker compose up -d
docker exec ollama nvidia-smi -L
```

{{% notice tip %}}
A safety net against forgetting: right after starting the instance, schedule an automatic shutdown, e.g.
`sudo shutdown -h +180` (3 hours). With the default "stop" shutdown behavior this stops the instance; it does not
terminate it.
{{% /notice %}}

### 2. Reset the lab data

```bash
docker compose down -v       # also deletes MySQL data, logs, the decision DB and the Ollama model volume
```

### 3. Remove everything

1. Terminate the EC2 instance (this deletes its root volume).
2. Delete the security group and the key pair if you created them only for this lab.
3. Delete any leftover EBS volumes or snapshots in the EC2 console.
