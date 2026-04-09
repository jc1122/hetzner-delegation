---
name: hetzner-delegation
description: 'Use when computational work should run remotely — "run this experiment", "train this model", "compute", "benchmark", "run this on ray", "delegate this", "offload this workload", "enqueue this batch", "submit this campaign". Covers experiments, training, benchmarks, data processing, queue batches, and remote project runs via the ray-hetzner environment. If dispatching a subagent, instruct it explicitly to use the hetzner-delegation skill and not execute the workload locally.'
---

# hetzner-delegation

> **Note:** This skill is implemented in the `ray-hetzner` repository (`~/projects/ray-hetzner`), which this skill directory mirrors. All scripts (`setup_head.sh`, `add_worker.sh`, etc.) and the queue backend (`metaopt/`) live there.

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
# Edit config.env to set your Hetzner SSH key name and other site-specific values
```

Required environment:

- authenticated `hcloud` CLI context
- local `jq`, `rsync`, and `ssh`
- a populated `~/projects/ray-hetzner/config.env`
- optional local `ray[default]` install for connectivity/smoke checks

`setup_head.sh` runs preflight checks automatically (SSH key, snapshot, network, quota) and will fail clearly if any prerequisite is missing.

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
./build_base_snapshot.sh      # only when no `ray-worker-base-*` snapshot exists yet
./setup_head.sh               # idempotent: creates head if absent, restarts Ray if head already exists
./add_worker.sh 1             # creates ray-worker-1 (suffix string, not a count)
./add_worker.sh 2             # creates ray-worker-2; each call creates one new server
```

`setup_head.sh` is idempotent — safe to rerun.
`add_worker.sh` is **NOT idempotent** — each call creates a new server with the given suffix.
Always run `./status.sh` first to check how many workers already exist before adding more.
They automatically clear stale SSH known_hosts entries when a server is reprovisioned at a reused IP.

### 3. Validate the cluster

```bash
cd ~/projects/ray-hetzner
# Run on the head node — Ray ports are firewall-blocked on the public interface by design
ssh root@"$RAY_HEAD_IP" "/opt/ray-env/bin/python3 /root/ray-hetzner/connect_test.py"
ssh root@"$RAY_HEAD_IP" "/opt/ray-env/bin/python3 /root/ray-hetzner/smoke_test.py"
```

To run from the laptop instead, open an SSH tunnel first:

```bash
ssh -L 10001:$RAY_HEAD_PRIVATE_IP:10001 root@"$RAY_HEAD_IP" -N &
RAY_ADDRESS=ray://localhost:10001 python3 connect_test.py
```

After validating the cluster, run any project-specific smoke test documented in the
project's `AGENTS.md` (e.g. `python3 scripts/features.py` for mit_bih_dg). This confirms
that code, data, and packages are correct on the target node — not just that Ray is up.

### 4. Sync code explicitly

```bash
cd ~/projects/ray-hetzner
./push_code.sh --no-data ~/projects/MyProject
```

Use `--no-data` by default. Only sync large data explicitly when the user asks for it.

> **Important — aorus is NOT a Hetzner node.** `push_code.sh` only syncs to cloud servers
> listed by `hcloud server list`. If the project's GPU work runs on aorus, always sync
> code and data to it separately:
>
> ```bash
> # Code (always when project changes)
> rsync -az --delete --exclude='.git' --exclude='__pycache__' --exclude='*.pyc' \
>       --exclude='data/' \
>       ~/projects/MyProject/ aorus:~/projects/MyProject/
>
> # Data (once, or when the dataset changes)
> rsync -az ~/projects/MyProject/data/ aorus:~/projects/MyProject/data/
> ```
>
> Aorus uses the `aorus` SSH alias (Tailscale, defined in `~/.ssh/config`).
> Both syncs use `rsync --delete` semantics so re-running is safe.

### 5. Run the workload

Submit the driver as a Ray job — this detaches from the SSH connection immediately and lets Ray manage the job lifecycle:

```bash
cd ~/projects/ray-hetzner
source config.env
ssh root@"$RAY_HEAD_IP" \
  "/opt/ray-env/bin/ray job submit \
    --address auto \
    --working-dir /root/MyProject \
    --entrypoint 'python path/to/script.py'"
```

The command prints a job ID (e.g. `raysubmit_abc123`). The SSH connection can drop — the job runs until completion.

Poll status or stream logs at any time:

```bash
ssh root@"$RAY_HEAD_IP" \
  "/opt/ray-env/bin/ray job status raysubmit_abc123"

ssh root@"$RAY_HEAD_IP" \
  "/opt/ray-env/bin/ray job logs --follow raysubmit_abc123"
```

> **When to use tmux instead:** only for long-running daemons you want to attach to interactively,
> such as `head_daemon.py`. For benchmark scripts and one-shot workloads, `ray job submit` is
> the correct pattern — it gives you job-ID tracking, structured logs, and no SSH dependency.

### 6. Collect logs and optional results

```bash
cd ~/projects/ray-hetzner
./collect_logs.sh raysubmit_abc123 --results /root/MyProject/results
```

Pass the job ID to get per-job stdout/stderr from the head. `--results` rsyncs the results directory from all Hetzner nodes.

> **Aorus results not included.** `collect_logs.sh` only fetches from `hcloud`-listed
> `ray-worker-*` nodes. If GPU tasks wrote results on aorus, fetch them separately:
>
> ```bash
> rsync -az aorus:~/projects/MyProject/results/ ~/projects/MyProject/results/aorus/
> ```

### 7. Cleanup only when requested

```bash
cd ~/projects/ray-hetzner
./remove_worker.sh ray-worker-1
./teardown_head.sh
```

## Sync Policy

- **Code sync to head/workers:** automatic and expected before direct delegation (`push_code.sh`)
- **Code sync to aorus:** always required separately — `push_code.sh` is blind to aorus
- **Data sync to head/workers:** explicit only; do not guess that local `data/` directories belong on the cluster
- **Data sync to aorus:** explicit and separate; aorus is never reached by `push_code.sh` or the queue workspace sync — pre-stage data at a fixed path before any GPU task runs there
- **Remote targets:** always use dedicated synced directories because `push_code.sh` uses `rsync --delete`
- **Results:** collect logs by default; collect full result trees when the user requests them

## Queue Delegation Workflow

Use this path when the request already fits the `metaopt` batch contract or the user explicitly asks for queue-backed campaign execution. Otherwise use direct delegation.

Queue mode requires a running head node. If `status.sh` does not show `ray-head`, complete direct delegation steps 1-3 first.

### 1. Enqueue the batch

```bash
cd ~/projects/ray-hetzner
python3 metaopt/enqueue_batch.py --manifest /path/to/batch.json \
    --queue-root /root/<project-name>/.ml-metaopt
```

The batch manifest is an immutable JSON document. Required fields: `version`, `campaign_id`, `iteration`, `batch_id`, `experiment`, `retry_policy.max_attempts`, `artifacts.code_artifact.uri`, `artifacts.data_manifest.uri`, `execution.entrypoint`. Do not silently pack large datasets into the code artifact; treat dataset movement as an explicit decision.

> Batch manifest schema is defined in `ml-metaoptimization/references/contracts.md` (Batch Manifest Contract section).

> **Aorus GPU tasks in queue batches**: the queue runner's workspace sync (`_sync_workspace_to_workers`)
> only reaches `hcloud`-listed `ray-worker-*` nodes — **aorus is never included**.
> Any Ray task with `resources={"aorus": 1}` runs on aorus using its pre-staged files,
> not the batch workspace. Ensure code and data are pre-staged on aorus at fixed paths
> before enqueueing, and write entrypoints to reference those fixed paths for GPU work.

### 2. Run the reconciler

```bash
cd ~/projects/ray-hetzner
python3 metaopt/head_daemon.py --queue-root /root/<project-name>/.ml-metaopt --once
```

`--once` runs a single reconciliation pass, recovering stale batches and processing one inbox item before it exits. Use the long-running mode (omit `--once`) when the user needs continuous processing. For concurrent multi-project execution, run one long-running daemon per project in a dedicated tmux session on the head, each with its own `--queue-root`.

### 3. Poll status

```bash
cd ~/projects/ray-hetzner
python3 metaopt/get_batch_status.py \
    --queue-root /root/<project-name>/.ml-metaopt \
    --batch-id batch-001
```

### 4. Fetch results

```bash
cd ~/projects/ray-hetzner
python3 metaopt/fetch_batch_results.py \
    --queue-root /root/<project-name>/.ml-metaopt \
    --batch-id batch-001
```

In the default remote mode, status and results are persisted on the head. Use the batch ID to retrieve them at any time after submission.

## Safety Rules

- Reuse the head by default.
- Do not perform destructive cleanup unless the user clearly asks for it.
- Do not silently sync large datasets.
- Make the chosen mode explicit: direct vs queue, reuse vs bootstrap, sync scope, cleanup scope.
- Use `ray-hetzner`'s documented scripts and queue commands instead of raw Hetzner fallbacks.
- **Multi-project queue isolation**: always pass `--queue-root /root/<project-name>/.ml-metaopt`
  to every metaopt queue command **including `head_daemon.py`**. Never rely on `METAOPT_REMOTE_QUEUE_ROOT` from `config.env`
  as a cross-project default — that variable reflects the last active campaign only.
  For concurrent multi-project runs, start a separate `head_daemon.py` per project (each with its own `--queue-root`)
  in dedicated tmux sessions on the head.
- **Worker provisioning**: run `./status.sh` before `add_worker.sh`. The script enforces
  `RAY_MAX_WORKERS=4` (account limit 5 total). Do not add workers beyond workload needs.

## Failure Handling

If prerequisites are missing for real execution, fail clearly and point to the blocked `ray-hetzner` step instead of inventing a fallback. Use dry-run/rehearsal behavior only when the user is planning or inspecting (`--dry-run`, `--allow-missing-prereqs`).
