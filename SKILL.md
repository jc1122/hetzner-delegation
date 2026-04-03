---
name: hetzner-delegation
description: 'Use when any computational work needs to be run — experiments, training, benchmarks, data processing, or scripts. Triggers on: "run experiment", "run this", "train", "compute", "benchmark", "offload", "delegate to Hetzner", "run on a server". If dispatching a subagent to do computational work, instruct it explicitly to use the hetzner-delegation skill and the Hetzner server worker.'
---

# hetzner-delegation

## Overview

Delegate heavy computational experiments to a short-lived Hetzner `cx53` server. The **local agent controls the server over SSH** — it issues commands, monitors output, and transfers files. No remote agent is spawned. The server is provisioned from a snapshot when one exists (saves setup time), otherwise from a base image. After the experiment completes, a new snapshot is created and the server is deleted.

**Core principle:** Always check for idle running servers before spawning a new one. Every session ends with: snapshot → delete. Never delete without snapshotting first.

## When Dispatching Subagents

If you delegate computational work to a subagent via the Agent tool, you MUST include this instruction in the subagent prompt:

> "Use the hetzner-delegation skill. Run this work on the Hetzner server worker — do not execute it locally."

Without this, the subagent will run the work locally by default. The instruction must be explicit.

## Prerequisites

```bash
# Install hcloud CLI
curl -sSL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz | tar xz
sudo mv hcloud /usr/local/bin/

# Authenticate (one-time)
export HCLOUD_TOKEN=<your-api-token>   # or set in ~/.config/hcloud/cli.toml

# Upload your SSH public key to Hetzner (one-time)
hcloud ssh-key create --name my-key --public-key-from-file ~/.ssh/id_rsa.pub
hcloud ssh-key list
```

## Full Lifecycle Pattern

### 1. Check for Idle Running Servers (FIRST — before provisioning)

```bash
SSH_OPTS="-o StrictHostKeyChecking=no -o ConnectTimeout=5"

# List all running servers
RUNNING=$(hcloud server list -o json | jq -r '.[] | select(.status=="running") | "\(.name) \(.public_net.ipv4.ip)"')

SERVER=""
IP=""

while IFS=' ' read -r name ip; do
  [ -z "$name" ] && continue
  # Check 1-min load average vs CPU count — idle if load < 50% of CPUs
  LOAD=$(ssh $SSH_OPTS root@"$ip" "cat /proc/loadavg" 2>/dev/null | awk '{print int($1)}')
  CPUS=$(ssh $SSH_OPTS root@"$ip" "nproc" 2>/dev/null)
  if [ -n "$LOAD" ] && [ -n "$CPUS" ] && [ "$LOAD" -lt $(( CPUS / 2 )) ]; then
    echo "Reusing idle server: $name ($ip) — load $LOAD / $CPUS CPUs"
    SERVER="$name"
    IP="$ip"
    break
  else
    echo "Server $name is busy (load $LOAD / $CPUS CPUs), skipping."
  fi
done <<< "$RUNNING"
```

If `SERVER` and `IP` are set after this block, skip steps 2–3 and go straight to step 4.

### 2. Find or Use Snapshot (only if no idle server found)

```bash
# List available snapshots
hcloud image list --type snapshot

# Pick the most recent suitable snapshot (by description pattern)
SNAPSHOT_ID=$(hcloud image list --type snapshot -o json | \
  jq -r 'sort_by(.created) | reverse | .[0].id // empty')

# If SNAPSHOT_ID is empty, fall back to base image
IMAGE=${SNAPSHOT_ID:-ubuntu-24.04}
```

### 2. Provision

```bash
SERVER="worker-$(date +%s)"
SSH_KEY=<your-key-name>    # from `hcloud ssh-key list`

hcloud server create \
  --name "$SERVER" \
  --type cx53 \
  --image "$IMAGE" \
  --ssh-key "$SSH_KEY" \
  --location nbg1 \        # nbg1 (Nuremberg), fsn1 (Falkenstein), hel1 (Helsinki), ash (Ashburn)
  --wait                   # blocks until action completes; SSH may need ~5s more
```

### 3. Wait for SSH

```bash
IP=$(hcloud server ip "$SERVER")

# --wait means the API action finished, not that sshd is listening yet
until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 root@"$IP" true 2>/dev/null; do
  sleep 2
done
```

### 4. Transfer Experiment

```bash
rsync -az --delete \
  -e "ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10" \
  ./experiment/ root@"$IP":/root/experiment/
```

### 5. Run Experiment

```bash
ssh -o StrictHostKeyChecking=no root@"$IP" bash << 'EOF'
  set -euo pipefail
  cd /root/experiment
  python3 main.py > output.txt 2>&1
EOF
```

### 6. Collect Results

```bash
rsync -az \
  -e "ssh -o StrictHostKeyChecking=no" \
  root@"$IP":/root/experiment/output/ ./results/
```

### 7. Snapshot (MANDATORY before deletion)

```bash
SNAP_DESC="worker-snapshot-$(date +%Y%m%d-%H%M)"
hcloud server create-image "$SERVER" \
  --type snapshot \
  --description "$SNAP_DESC" \
  --wait
echo "Snapshot '$SNAP_DESC' created."
```

### 8. Delete Server (MANDATORY)

```bash
hcloud server delete "$SERVER"
```

## Server Types

| Type    | vCPU | RAM   | Use case                        |
|---------|------|-------|---------------------------------|
| **cx53**| **20**| **40 GB** | **Default — heavy experiments** |
| cx43    | 8    | 16 GB | Lighter workloads               |
| cx33    | 4    | 8 GB  | Dev/test runs                   |
| ccx53   | 16   | 64 GB | CPU-optimized (dedicated)       |

Run `hcloud server-type list` for current pricing and availability.

## Complete Script Template

```bash
#!/usr/bin/env bash
set -euo pipefail

SSH_KEY="${HETZNER_SSH_KEY:?set HETZNER_SSH_KEY}"
SSH_OPTS="-o StrictHostKeyChecking=no -o ConnectTimeout=15"
SNAP_DESC="worker-snapshot-$(date +%Y%m%d-%H%M)"
SERVER=""
IP=""
SPAWNED=false

snapshot_and_delete() {
  echo "Creating snapshot '$SNAP_DESC'..."
  hcloud server create-image "$SERVER" --type snapshot --description "$SNAP_DESC" --wait \
    && echo "Snapshot created." \
    || echo "WARNING: snapshot failed — not deleting server to avoid data loss"
  hcloud server delete "$SERVER" 2>/dev/null && echo "Server deleted." || true
}
trap snapshot_and_delete EXIT INT TERM

# 1. Check for idle running servers
echo "Checking for idle running servers..."
while IFS=' ' read -r name ip; do
  [ -z "$name" ] && continue
  LOAD=$(ssh $SSH_OPTS root@"$ip" "cat /proc/loadavg" 2>/dev/null | awk '{print int($1)}') || continue
  CPUS=$(ssh $SSH_OPTS root@"$ip" "nproc" 2>/dev/null) || continue
  if [ "$LOAD" -lt $(( CPUS / 2 )) ]; then
    echo "Reusing idle server: $name ($ip) — load $LOAD/$CPUS"
    SERVER="$name"; IP="$ip"
    break
  else
    echo "Server $name busy (load $LOAD/$CPUS), skipping."
  fi
done < <(hcloud server list -o json | jq -r '.[] | select(.status=="running") | "\(.name) \(.public_net.ipv4.ip)"')

# 2. Provision if no idle server found
if [ -z "$SERVER" ]; then
  SERVER="worker-$(date +%s)"
  SPAWNED=true
  SNAPSHOT_ID=$(hcloud image list --type snapshot -o json | \
    jq -r 'sort_by(.created) | reverse | .[0].id // empty')
  IMAGE=${SNAPSHOT_ID:-ubuntu-24.04}
  echo "No idle server found. Provisioning $SERVER from $IMAGE..."
  hcloud server create --name "$SERVER" --type cx53 --image "$IMAGE" \
    --ssh-key "$SSH_KEY" --location nbg1 --wait
  IP=$(hcloud server ip "$SERVER")
  echo "Server at $IP, waiting for SSH..."
  until ssh $SSH_OPTS root@"$IP" true 2>/dev/null; do sleep 2; done
fi

# 3. Transfer experiment
echo "Transferring experiment..."
rsync -az --delete -e "ssh $SSH_OPTS" ./experiment/ root@"$IP":/root/experiment/

# 4. Run experiment
echo "Running experiment..."
ssh $SSH_OPTS root@"$IP" "cd /root/experiment && bash run.sh"

# 5. Collect results
echo "Collecting results..."
rsync -az -e "ssh $SSH_OPTS" root@"$IP":/root/experiment/output/ ./results/

echo "Done. Snapshot and cleanup will run on exit."
```

`trap snapshot_and_delete EXIT` runs even on failure — server is never left running and state is preserved in a snapshot.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Spawning new server when idle one exists | Always check running servers first (step 1) |
| Deleting server without snapshot | Always snapshot first; trap guards this |
| SSH before server is ready | Use the SSH poll loop after `--wait` |
| Host key prompt blocks first SSH | `-o StrictHostKeyChecking=no` everywhere |
| Server left running after failure | `trap` on EXIT INT TERM |
| Name collision on re-run | `$(date +%s)` suffix |
| Wrong IP (IPv6) | `hcloud server ip` returns IPv4 by default |
| No suitable snapshot found | Script falls back to `ubuntu-24.04` automatically |
| Snapshot of wrong server | Pass `$SERVER` explicitly to `create-image` |

## How the Local Agent Controls the Server

The local Claude agent drives everything over SSH — it does NOT spawn a remote agent instance. The local agent:

1. Issues shell commands via SSH to run steps on the server
2. Streams output back for monitoring and logging
3. Transfers files with rsync
4. Decides when to collect results and shut down

```bash
# Run a command and stream output locally
ssh $SSH_OPTS root@"$IP" "cd /root/experiment && python3 main.py" | tee results/run.log

# Check a long-running job's progress
ssh $SSH_OPTS root@"$IP" "tail -f /root/experiment/output.txt"

# Run a background job and poll for completion
ssh $SSH_OPTS root@"$IP" "nohup bash /root/experiment/run.sh > /root/experiment/output.txt 2>&1 &"
# ... poll:
ssh $SSH_OPTS root@"$IP" "tail -1 /root/experiment/output.txt"
```
