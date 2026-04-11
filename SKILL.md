---
name: hetzner-delegation
description: 'Use when computational work should run remotely through the ray-hetzner environment — experiments, training, benchmarks, data processing, queue batches, or remote project runs. Triggers on: "delegate this", "run this on ray", "run this on ray-hetzner", "enqueue this batch", "submit this campaign", "offload this workload". If dispatching a subagent, instruct it explicitly to use the hetzner-delegation skill and not execute the workload locally.'
---

# hetzner-delegation

> **Note:** This skill is the agent-facing contract for the `ray-hetzner` backend at `~/projects/ray-hetzner`. The backend owns the executable scripts, queue runtime, autoscaler config, and operator docs.

## Overview

Delegate computational work through `ray-hetzner`.

This skill is delegation-first. It may refresh the Aorus control plane, submit queue-backed work, run one-off Ray jobs, inspect status, and fetch results. It is not a generic Hetzner administration manual.

Use the backend docs as source of truth for commands:

- `~/projects/ray-hetzner/README.md`
- `~/projects/ray-hetzner/docs/OPERATIONS.md`

## Execution Paths

- **Queue delegation (primary):** use `metaopt/` queue commands for batch and campaign workflows.
- **Direct delegation (secondary):** use `submit_ray_job.sh` for one-off scripts that do not fit the queue contract.

Choose queue mode by default when the work can be expressed as a batch manifest or campaign submission. Use direct mode for ad hoc scripts, smoke runs, or operator-driven one-off jobs.

## When Dispatching Subagents

If you delegate compute work to a subagent, you MUST include an explicit instruction to use this skill. Without this, the subagent will run the work locally by default.

> "Use the hetzner-delegation skill. Delegate this work through ray-hetzner and do not execute it locally."

## Prerequisites

```bash
cd ~/projects/ray-hetzner
cp config.env.example config.env
```

Required environment:

- authenticated `hcloud` CLI context
- local `ssh`, `rsync`, and `jq`
- Aorus reachable over Tailscale
- a populated `~/projects/ray-hetzner/config.env`

Primary config variables:

- `AORUS_TAILSCALE_IP`
- `AORUS_SSH_USER=jakub`
- `AORUS_RAY_VENV=/home/jakub/ray-venv`
- `TAILSCALE_AUTH_KEY`
- `HETZNER_SSH_KEY`
- `METAOPT_REMOTE_REPO_ROOT=/home/jakub/ray-hetzner`

Compatibility variables such as `RAY_HEAD_IP` may still exist, but new workflow logic should treat `AORUS_TAILSCALE_IP` as canonical.

## Canonical Bootstrap

Refresh the permanent control plane before real execution when needed:

```bash
cd ~/projects/ray-hetzner
./setup_aorus.sh
./status.sh
```

Rehearse the bootstrap without contacting Aorus or mutating `config.env`:

```bash
cd ~/projects/ray-hetzner
./setup_aorus.sh --dry-run
```

`setup_aorus.sh` is the supported bootstrap entrypoint. It refreshes the Ray head on Aorus, installs or updates the queue daemon service, and keeps compatibility variables in sync.

`status.sh` is the supported state check. It should be used to confirm:

- Aorus head identity and connectivity
- Ray cluster status from the head
- Hetzner worker inventory
- Hetzner network state when present

## Worker Capacity Model

Hetzner workers are disposable capacity, not the control plane.

Supported scaling path:

- `cluster.yaml` defines the Ray cluster
- `metaopt/hetzner_node_provider.py` bridges Ray autoscaler to Hetzner
- workers boot from the latest `ray-worker-base-*` snapshot
- workers join over Tailscale

Before the first autoscaled worker job, a base snapshot must exist:

```bash
cd ~/projects/ray-hetzner
./build_base_snapshot.sh
```

Re-run `build_base_snapshot.sh` only when worker dependencies change. The snapshot is reused across runs.

Do not use the removed manual head or worker lifecycle as a fallback.

## Queue Delegation Workflow

This is the primary execution path.

Always use an explicit per-project queue root:

```bash
--queue-root /home/jakub/projects/<project>/.ml-metaopt
```

Do not rely on `METAOPT_REMOTE_QUEUE_ROOT` as a durable multi-project default.

### 1. Bootstrap or inspect the control plane

```bash
cd ~/projects/ray-hetzner
./setup_aorus.sh --dry-run
./setup_aorus.sh
./status.sh
```

Skip the real bootstrap if `status.sh` already shows Aorus is healthy and the queue daemon state is adequate for the requested work.

### 2. Enqueue the batch

```bash
cd ~/projects/ray-hetzner
python3 metaopt/enqueue_batch.py \
  --manifest /path/to/batch.json \
  --queue-root /home/jakub/projects/<project>/.ml-metaopt
```

Dry-run rehearsal:

```bash
cd ~/projects/ray-hetzner
python3 metaopt/enqueue_batch.py \
  --manifest /path/to/batch.json \
  --queue-root /home/jakub/projects/<project>/.ml-metaopt \
  --dry-run
```

Expected remote-client behavior:

- code artifacts are copied to Aorus
- the manifest is rewritten to point at Aorus-side artifact paths
- authoritative queue state lives on Aorus

### 3. Reconcile the queue

One-shot on Aorus:

```bash
cd ~/projects/ray-hetzner
source config.env
ssh "$AORUS_SSH_USER@$AORUS_TAILSCALE_IP" \
  "cd /home/jakub/ray-hetzner && \
   METAOPT_BACKEND_DISABLE_REMOTE=1 python3 -m metaopt.head_daemon \
   --queue-root /home/jakub/projects/<project>/.ml-metaopt \
   --once"
```

Continuous multi-project mode is managed by the Aorus user service installed by `setup_aorus.sh`. Configure watched roots with `METAOPT_QUEUE_ROOTS` in `config.env` (colon-separated). Re-run `setup_aorus.sh` after changing `METAOPT_QUEUE_ROOTS` to push the updated list to Aorus **and restart the shared daemon** — the daemon will not pick up the new root until then. Do not launch per-project long-running daemons for normal operation; keep one shared watcher and isolate projects by queue root.

### 4. Poll status

```bash
cd ~/projects/ray-hetzner
python3 metaopt/get_batch_status.py \
  --queue-root /home/jakub/projects/<project>/.ml-metaopt \
  --batch-id batch-001
```

Dry-run rehearsal:

```bash
cd ~/projects/ray-hetzner
python3 metaopt/get_batch_status.py \
  --queue-root /home/jakub/projects/<project>/.ml-metaopt \
  --batch-id batch-001 \
  --dry-run
```

### 5. Fetch results

```bash
cd ~/projects/ray-hetzner
python3 metaopt/fetch_batch_results.py \
  --queue-root /home/jakub/projects/<project>/.ml-metaopt \
  --batch-id batch-001
```

Dry-run rehearsal:

```bash
cd ~/projects/ray-hetzner
python3 metaopt/fetch_batch_results.py \
  --queue-root /home/jakub/projects/<project>/.ml-metaopt \
  --batch-id batch-001 \
  --dry-run
```

Use `ray job status` or `ray job logs` on Aorus only when diagnosing an active job beyond what queue state already reports.

## Direct Delegation Workflow

Use this path for one-off runs that do not fit the queue contract.

### 1. Refresh or inspect the control plane

```bash
cd ~/projects/ray-hetzner
./setup_aorus.sh --dry-run
./setup_aorus.sh
./status.sh
```

### 2. Sync code explicitly to a dedicated remote path

```bash
rsync -az --delete \
  --exclude='.git' \
  --exclude='__pycache__' \
  ~/projects/MyProject/ \
  aorus:~/projects/MyProject/
```

Use dedicated remote directories because `rsync --delete` is destructive outside isolated targets. Never sync large datasets implicitly.

### 3. Submit the run through Ray Jobs

```bash
cd ~/projects/ray-hetzner
./submit_ray_job.sh \
  --working-dir /home/jakub/projects/MyProject \
  --entrypoint "python3 path/to/script.py"
```

Dry-run rehearsal:

```bash
cd ~/projects/ray-hetzner
./submit_ray_job.sh \
  --dry-run \
  --working-dir /home/jakub/projects/MyProject \
  --entrypoint "python3 path/to/script.py"
```

The helper returns a Ray job ID during real execution. Prefer Ray Jobs over interactive terminal sessions for one-shot work.

### 4. Track logs and results

Use backend-documented Ray Jobs commands for active-job diagnostics. Fetch result trees explicitly with `rsync` when the user asks for them.

## Sync and Data Policy

- **Code sync:** explicit and targeted; use dedicated remote paths.
- **Queue artifacts:** let queue mode move code artifacts instead of inventing a second sync path.
- **Large datasets:** never sync implicitly.
- **Results:** always make retrieval explicit.
- **Aorus-specific files:** stage them separately when a workflow depends on fixed paths on Aorus.

## Safety Rules

- Reuse the Aorus head by default.
- Prefer queue mode when the work fits the queue contract.
- Do not perform destructive cleanup unless the user clearly asks for it.
- Do not silently sync large datasets.
- Make the chosen mode explicit: queue vs direct, bootstrap vs reuse, sync scope, and cleanup scope.
- Use `ray-hetzner`'s documented primary surface, not removed lifecycle scripts.
- Always pass an explicit `--queue-root` for queue commands.

## Legacy Boundary

The old manual Hetzner head and worker lifecycle has been removed from the backend. Do not present it as a fallback, and do not recreate it in this skill.

## Failure Handling

If prerequisites are missing for real execution, fail clearly and point to the blocked `ray-hetzner` step instead of inventing a fallback.

Use dry-run or rehearsal behavior only when the user is explicitly planning, inspecting, or validating safety before a real remote operation.
