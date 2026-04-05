---
name: hetzner-delegation
description: 'Use when computational work should run remotely — "run this experiment", "train this model", "compute", "benchmark", "run this on ray", "delegate this", "offload this workload", "enqueue this batch", "submit this campaign". Covers experiments, training, benchmarks, data processing, queue batches, and remote project runs via the ray-hetzner environment. If dispatching a subagent, instruct it explicitly to use the hetzner-delegation skill and not execute the workload locally.'
---

# hetzner-delegation

## Overview

Delegate computational work through the `ray-hetzner` repository. This skill is delegation-first: it may reuse or bootstrap the cluster, sync code, run work directly on the Ray cluster, or submit queue-backed `metaopt` batches when the request fits that workflow.

All cluster scripts and queue commands live in the `ray-hetzner` repository (`~/projects/ray-hetzner`). Start from its `README.md` and `docs/` for command discovery.

The skill is not a generic Hetzner administration manual. It should only perform the cluster lifecycle steps needed to make delegation succeed.

## Execution Paths

- **Direct delegation (default):** use the `ray-hetzner` cluster scripts for arbitrary project runs.
- **Queue delegation (specialized):** use the `metaopt` queue commands for batch/campaign workflows.

Queue mode should only be chosen when the request clearly matches a batch-manifest or queue-backed campaign shape. Otherwise use direct delegation.

## When Dispatching Subagents

If you delegate compute work to a subagent, you MUST include an explicit instruction to use this skill. Without this, the subagent will run the work locally by default.

> "Use the hetzner-delegation skill. Delegate this work through ray-hetzner and do not execute it locally."

## Prerequisites

```bash
cd ~/projects/ray-hetzner
cp config.env.example config.env
# Edit config.env to set your Hetzner token, SSH key, and other site-specific values
```

Required environment:

- authenticated `hcloud` CLI context
- local `jq`, `rsync`, and `ssh`
- a populated `~/projects/ray-hetzner/config.env`
- optional local `ray[default]` install for connectivity/smoke checks

## Direct Delegation Workflow

### 1. Check cluster status first

```bash
cd ~/projects/ray-hetzner
./status.sh
```

If `status.sh` already shows `ray-head` and the needed `ray-worker-*` servers, reuse the existing cluster and skip to step 3.

### 2. Bootstrap the cluster (only when needed)

```bash
cd ~/projects/ray-hetzner
./build_base_snapshot.sh      # only when `hcloud image list -t snapshot -l type=ray-worker-base` is empty
./setup_head.sh
./add_worker.sh 1             # add more workers only when the workload needs them
```

### 3. Validate the cluster

```bash
cd ~/projects/ray-hetzner
python3 connect_test.py        # verifies SSH and Ray connectivity
python3 smoke_test.py          # runs a minimal remote task
```

### 4. Sync code explicitly

```bash
cd ~/projects/ray-hetzner
./push_code.sh --no-data ~/projects/MyProject
```

Use `--no-data` by default. Only sync large data explicitly when the user asks for it.

### 5. Run the workload

```bash
cd ~/projects/ray-hetzner
source config.env
ssh root@"$RAY_HEAD_IP"
```

Then on the head node:

```bash
tmux new -s run
python /root/MyProject/path/to/script.py
```

### 6. Collect logs and optional results

```bash
cd ~/projects/ray-hetzner
./collect_logs.sh --results /root/MyProject/results
```

### 7. Cleanup only when requested

```bash
cd ~/projects/ray-hetzner
./remove_worker.sh ray-worker-1
./teardown_head.sh
```

## Sync Policy

- **Code sync:** automatic and expected before direct delegation
- **Data sync:** explicit only; do not guess that local `data/` directories belong on the cluster
- **Remote targets:** always use dedicated synced directories because `push_code.sh` uses `rsync --delete`
- **Results:** collect logs by default; collect full result trees when the user requests them
