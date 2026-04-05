---
name: hetzner-delegation
description: 'Use when computational work should run remotely — "run this experiment", "train this model", "compute", "benchmark", "run this on ray", "delegate this", "offload this workload", "enqueue this batch", "submit this campaign". Covers experiments, training, benchmarks, data processing, queue batches, and remote project runs via the ray-hetzner environment. If dispatching a subagent, instruct it explicitly to use the hetzner-delegation skill and not execute the workload locally.'
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

If you delegate compute work to a subagent, you MUST include an explicit instruction to use this skill. Without this, the subagent will run the work locally by default.

> "Use the hetzner-delegation skill. Delegate this work through ray-hetzner and do not execute it locally."
