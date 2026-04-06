# Ray Hetzner Delegation Rewrite Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite `hetzner-delegation` so it delegates work through `ray-hetzner`'s direct-cluster and queue workflows instead of the legacy one-server Hetzner SSH lifecycle.

**Architecture:** Keep one rewritten skill. Make it delegation-first: it may reuse/bootstrap the `ray-hetzner` cluster to make work delegation succeed, but it should not become a generic Hetzner administration manual. Use two explicit execution paths in the skill text: direct delegation by default, queue delegation for `metaopt`-shaped requests.

**Tech Stack:** Markdown skill docs (`SKILL.md`, `README.md`, `CLAUDE.md`), Git, existing `ray-hetzner` shell/Python commands and docs

---

## File Structure

- **Modify:** `SKILL.md`
  - Replace the stale front matter, overview, prerequisites, lifecycle, examples, and safeguards with a `ray-hetzner`-native delegation workflow.
- **Modify:** `README.md`
  - Rewrite the repository summary so it describes the rewritten skill and its `ray-hetzner` focus.
- **Modify:** `CLAUDE.md`
  - Refresh the internal handoff text so it mirrors the new skill intent and sync instructions.
- **Reference:** `docs/superpowers/specs/2026-04-05-ray-hetzner-delegation-design.md`
  - Use the approved design as the source of truth while implementing.

### Task 1: Rewrite the skill identity and routing contract

**Files:**
- Modify: `SKILL.md`
- Reference: `docs/superpowers/specs/2026-04-05-ray-hetzner-delegation-design.md`
- Verify: `SKILL.md`

- [ ] **Step 1: Capture the legacy baseline that must disappear**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
rg -n "short-lived Hetzner `cx53` server|snapshot → delete|No remote agent is spawned|Full Lifecycle Pattern" SKILL.md
```

Expected: `rg` prints matches from the stale one-server skill text.

- [ ] **Step 2: Replace the front matter and opening sections with the new delegation-first contract**

Update the top of `SKILL.md` so the opening block looks like:

```md
---
name: hetzner-delegation
description: 'Use when work should be offloaded to the ray-hetzner environment — experiments, training, benchmarks, data processing, queue batches, or remote project runs. Triggers on: "delegate this", "run this on ray", "run this on ray-hetzner", "enqueue this batch", "submit this campaign", "offload this workload". If dispatching a subagent, instruct it explicitly to use the hetzner-delegation skill and not execute the workload locally.'
---

# hetzner-delegation

## Overview

Delegate computational work through the `ray-hetzner` repository. This skill is delegation-first: it may reuse or bootstrap the cluster, sync code, run work directly on the Ray cluster, or submit queue-backed `metaopt` batches when the request fits that workflow.

The skill is not a generic Hetzner administration manual. It should only perform the cluster lifecycle steps needed to make delegation succeed.

## Execution Paths

- **Direct delegation (default):** use the `ray-hetzner` cluster scripts for arbitrary project runs.
- **Queue delegation (specialized):** use the `metaopt` queue commands for batch/campaign workflows.

Queue mode should only be chosen when the request clearly matches a batch-manifest or queue-backed campaign shape. Otherwise use direct delegation.

## When Dispatching Subagents

If you delegate compute work to a subagent, include:

> "Use the hetzner-delegation skill. Delegate this work through ray-hetzner and do not execute it locally."
```

- [ ] **Step 3: Verify the stale identity is gone and the new routing language is present**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
rg -n "short-lived Hetzner `cx53` server|snapshot → delete|Full Lifecycle Pattern" SKILL.md
rg -n "ray-hetzner|Direct delegation|Queue delegation|delegation-first" SKILL.md
```

Expected:
- first command returns no matches
- second command returns matches in the rewritten overview/routing sections

- [ ] **Step 4: Commit the routing rewrite**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
git add SKILL.md
git commit -m "docs: rewrite skill contract around ray-hetzner"
```

Expected: one commit containing the front matter/overview/routing rewrite.

### Task 2: Replace the legacy server lifecycle with direct delegation workflow

**Files:**
- Modify: `SKILL.md`
- Reference: `/home/jakub/projects/ray-hetzner/README.md`
- Reference: `/home/jakub/projects/ray-hetzner/docs/OPERATIONS.md`
- Verify: `SKILL.md`

- [ ] **Step 1: Capture the stale lifecycle sections that must be removed**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
rg -n "Check for Idle Running Servers|Find or Use Snapshot|Provision|Wait for SSH|Snapshot \\(MANDATORY before deletion\\)|Delete Server \\(MANDATORY\\)|Server Types|Complete Script Template" SKILL.md
```

Expected: the command prints the current one-server lifecycle headings.

- [ ] **Step 2: Replace the prerequisites and direct-mode workflow with `ray-hetzner` commands**

Rewrite the lifecycle portion of `SKILL.md` so it includes a direct-delegation section shaped like:

````md
## Prerequisites

```bash
cd ~/projects/ray-hetzner
cp config.env.example config.env
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

### 2. Reuse or bootstrap the cluster

```bash
cd ~/projects/ray-hetzner
./build_base_snapshot.sh      # only when the base snapshot is still missing
./setup_head.sh
./add_worker.sh 1             # add more workers only when the workload needs them
```

### 3. Sync code explicitly

```bash
cd ~/projects/ray-hetzner
./push_code.sh --no-data ~/projects/MyProject
```

Use `--no-data` by default. Only sync large data explicitly when the user asks for it.

### 4. Run the workload

```bash
ssh root@"$RAY_HEAD_IP"
tmux new -s run
python /root/MyProject/path/to/script.py
```

### 5. Collect logs and optional results

```bash
cd ~/projects/ray-hetzner
./collect_logs.sh --results /root/MyProject/results
```

### 6. Cleanup only when requested

```bash
cd ~/projects/ray-hetzner
./remove_worker.sh ray-worker-1
./teardown_head.sh
```
````

- [ ] **Step 3: Add explicit sync/defaults language next to the direct workflow**

Add this policy text immediately after the direct workflow:

```md
## Sync Policy

- **Code sync:** automatic and expected before direct delegation
- **Data sync:** explicit only; do not guess that local `data/` directories belong on the cluster
- **Remote targets:** always use dedicated synced directories because `push_code.sh` uses `rsync --delete`
- **Results:** collect logs by default; collect full result trees when the user requests them
```

- [ ] **Step 4: Verify the skill now describes direct `ray-hetzner` delegation instead of the raw Hetzner lifecycle**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
rg -n "setup_head.sh|add_worker.sh|push_code.sh|collect_logs.sh|teardown_head.sh|--no-data" SKILL.md
rg -n "Server Types|Complete Script Template|Delete Server \\(MANDATORY\\)" SKILL.md
```

Expected:
- first command returns multiple matches
- second command returns no matches

- [ ] **Step 5: Commit the direct-delegation rewrite**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
git add SKILL.md
git commit -m "docs: replace legacy server lifecycle with ray delegation flow"
```

Expected: one commit containing the direct-mode rewrite.

### Task 3: Add queue delegation, safety rules, and failure behavior

**Files:**
- Modify: `SKILL.md`
- Reference: `/home/jakub/projects/ray-hetzner/README.md`
- Verify: `SKILL.md`

- [ ] **Step 1: Confirm queue-mode content is still missing before adding it**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
rg -n "metaopt|enqueue_batch.py|head_daemon.py|get_batch_status.py|fetch_batch_results.py" SKILL.md
```

Expected: no matches yet, or only incidental references that are clearly incomplete.

- [ ] **Step 2: Add the queue-mode section and routing cues**

Insert a queue-delegation section like:

````md
## Queue Delegation Workflow

Use this path when the request already fits the `metaopt` batch contract or the user is explicitly asking for queue-backed campaign execution.

### 1. Enqueue the batch

```bash
cd ~/projects/ray-hetzner
python3 metaopt/enqueue_batch.py --manifest /path/to/batch.json
```

### 2. Run or reuse the daemon

```bash
cd ~/projects/ray-hetzner
python3 metaopt/head_daemon.py --once
```

### 3. Poll status

```bash
cd ~/projects/ray-hetzner
python3 metaopt/get_batch_status.py --batch-id batch-001
```

### 4. Fetch results

```bash
cd ~/projects/ray-hetzner
python3 metaopt/fetch_batch_results.py --batch-id batch-001
```

Queue mode uses immutable code artifacts and persisted status/results. It should not silently pack large datasets into the artifact; treat dataset movement as an explicit decision.
````

- [ ] **Step 3: Add safety and failure-handling rules near the end of the skill**

Append a rules section shaped like:

```md
## Safety Rules

- Reuse the head by default.
- Do not perform destructive cleanup unless the user clearly asks for it.
- Do not silently sync large datasets.
- Make the chosen mode explicit: direct vs queue, reuse vs bootstrap, sync scope, cleanup scope.
- Use `ray-hetzner`'s documented scripts and queue commands instead of raw Hetzner fallbacks.

## Failure Handling

If prerequisites are missing for real execution, fail clearly and point to the blocked `ray-hetzner` step instead of inventing a fallback. Use dry-run/rehearsal behavior only when the user is planning or inspecting.
```

- [ ] **Step 4: Verify queue mode and safety language are now explicit**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
rg -n "enqueue_batch.py|head_daemon.py|get_batch_status.py|fetch_batch_results.py|Queue Delegation Workflow" SKILL.md
rg -n "Reuse the head by default|Do not silently sync large datasets|Failure Handling" SKILL.md
```

Expected: both commands return matches in the new sections.

- [ ] **Step 5: Commit the queue/safety rewrite**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
git add SKILL.md
git commit -m "docs: add queue delegation and safety rules"
```

Expected: one commit containing the queue-mode and safeguards text.

### Task 4: Refresh mirrored repo docs and publish

**Files:**
- Modify: `README.md`
- Modify: `CLAUDE.md`
- Verify: `README.md`
- Verify: `CLAUDE.md`

- [ ] **Step 1: Replace the stale repo summary in `README.md`**

Rewrite `README.md` to a concise mirror description like:

```md
# hetzner-delegation

Mirror repository for the `hetzner-delegation` skill.

The skill now delegates work through `ray-hetzner` rather than the legacy one-server Hetzner SSH lifecycle.

Primary paths:

- direct Ray-cluster delegation for arbitrary project runs
- queue-backed delegation for `metaopt` batch workflows

Design and planning docs live under `docs/superpowers/`.
```

- [ ] **Step 2: Rewrite `CLAUDE.md` so it mirrors the new skill reality**

Replace the stale narrative with a concise handoff block like:

````md
# hetzner-delegation

This repository mirrors the `hetzner-delegation` skill for version control.

Current direction:

- the old single-server Hetzner flow is obsolete
- the skill is being rewritten around `~/projects/ray-hetzner`
- default behavior is delegation-first direct cluster execution
- queue mode is reserved for `metaopt`-shaped work
- automatic code sync is acceptable; large data sync must stay explicit

Sync the live skill back into this repository after editing:

```bash
cp ~/.claude/skills/hetzner-delegation/SKILL.md ~/projects/hetzner-delegation/SKILL.md
```
````

- [ ] **Step 3: Verify the mirror docs no longer describe the stale flow**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
rg -n "short-lived Hetzner|snapshotting, and deleting|cx11|HCLOUD_TOKEN" README.md CLAUDE.md
rg -n "ray-hetzner|queue-backed|delegation-first|single-server Hetzner flow is obsolete" README.md CLAUDE.md
```

Expected:
- first command returns no stale workflow claims
- second command returns matches in both files

- [ ] **Step 4: Run final repository verification and publish**

Run:

```bash
cd /home/jakub/projects/hetzner-delegation
git diff --check
git --no-pager diff --stat
git add SKILL.md README.md CLAUDE.md
git commit -m "docs: rewrite hetzner-delegation around ray-hetzner"
git push origin master
```

Expected:
- `git diff --check` prints nothing
- final commit succeeds
- push to `origin/master` succeeds
