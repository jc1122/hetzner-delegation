# Ray Hetzner Delegation Rewrite Design

## Problem

`hetzner-delegation` is stale. It currently documents a one-server Hetzner SSH workflow, while the real delegation infrastructure now lives in `ray-hetzner` and supports a broader, safer lifecycle:

- reusable Ray head + scalable workers
- direct project delegation through the cluster scripts
- queue-backed batch delegation through `metaopt/`
- explicit dry-run and permissive rehearsal modes

The skill should be rewritten so its behavior matches the current `ray-hetzner` repository instead of the legacy single-server pattern.

## Goals

1. Make `hetzner-delegation` a **delegation-first** skill for `ray-hetzner`.
2. Allow the skill to perform the cluster setup work needed for delegation to succeed.
3. Support two execution paths:
   - direct delegation for arbitrary project runs
   - queue delegation for `metaopt` batch workflows
4. Keep code sync automatic, data sync explicit, and result collection explicit.
5. Preserve behavior-safe defaults: reuse where possible, avoid destructive cleanup unless requested.

## Non-Goals

1. Preserve the legacy one-server SSH lifecycle.
2. Turn the skill into a generic cluster administration manual.
3. Auto-sync large datasets without explicit user intent.
4. Hide `ray-hetzner` prerequisites behind unsupported raw Hetzner CLI fallbacks.

## Top-Level Skill Contract

The rewritten skill should answer the user intent:

> Offload this work to the `ray-hetzner` environment and carry it through to a usable result.

It remains a delegation skill, not an infrastructure skill. Cluster lifecycle actions are included only when they are required to make delegation succeed.

## Architecture

The skill should be organized around three responsibilities.

### 1. Work-shape routing

The skill decides whether the request is:

- a **direct delegation** request for an arbitrary project/script/run, or
- a **queue delegation** request that already fits the `metaopt` batch contract.

Default routing should prefer direct delegation. Queue mode should be selected only when the request clearly matches a batch/campaign workflow or already provides a manifest-oriented task.

### 2. Cluster-readiness orchestration

The skill checks the current `ray-hetzner` state first and then performs only the setup needed to delegate work:

- verify config/prerequisites
- reuse an existing head when available
- bootstrap missing required lifecycle steps through the documented scripts
- add workers when the requested work needs more capacity
- push code before execution

This readiness layer must use the existing `ray-hetzner` commands and documented semantics rather than raw Hetzner control flow.

### 3. Delegation closure

The skill should finish the operational loop:

- run or submit the workload
- monitor progress
- collect logs/results
- perform opt-in cleanup when the user requests it

## Execution Modes

### Direct delegation mode

This is the default user path for requests like:

- run this project remotely
- offload this script
- execute this experiment on Ray

Expected flow:

1. Check `ray-hetzner` readiness.
2. Reuse or bootstrap the cluster.
3. Sync the relevant project directories.
4. Run the workload through the documented project/Ray flow.
5. Collect logs and any requested results.

### Queue delegation mode

This is the specialized path for requests like:

- enqueue this batch manifest
- run this campaign through the queue backend
- poll/fetch queue-backed results

Expected flow:

1. Ensure the head-side queue runtime is reachable.
2. Enqueue the immutable batch artifact.
3. Start or reuse the head daemon as needed.
4. Poll status and fetch persisted results.

## Sync Policy

### Code sync

Code sync should be automatic and should use the existing `ray-hetzner` sync flow into dedicated remote paths. The skill should not sync into shared directories when `rsync --delete` is in play.

If a workflow depends on multiple local repositories, the skill should sync each one explicitly rather than guessing.

### Data sync

Data sync should be explicit. The skill should not assume that a local `data/` tree belongs on the cluster or that it is small enough to move safely.

Default policy:

- code sync: automatic
- data sync: opt-in

### Results sync

The skill should always make log/result retrieval explicit. It should:

- always surface logs/status needed to understand success or failure
- collect full results when the user requests them
- use queue result pointers naturally in queue mode

## Defaults and Safety Rules

1. Preserve the head by default.
2. Avoid destructive cleanup unless explicitly requested.
3. Prefer reuse over churn.
4. Fail clearly when prerequisites are missing for real execution.
5. Use dry-run/rehearsal behavior only when the user is asking to plan or inspect.
6. Make the chosen mode explicit in the response: direct vs queue, reuse vs bootstrap, sync scope, and cleanup behavior.

## Failure Handling

The rewritten skill should lean on the safety model already present in `ray-hetzner`:

- use documented scripts and queue commands
- surface their actionable failures rather than inventing silent fallbacks
- avoid unsupported raw Hetzner control paths

If prerequisites are missing, the skill should explain what blocked delegation and which `ray-hetzner` step is needed next.

## Documentation and Examples

The rewritten skill content should replace the legacy one-server examples with `ray-hetzner`-native examples for:

1. direct project delegation
2. queue-backed batch delegation
3. explicit code sync vs data sync choices
4. reuse/default cleanup behavior

The old single-server lifecycle should be removed, not retained as an alternate path.

## Verification Expectations

The implementation should be verified by checking that:

1. the skill description and triggers reflect `ray-hetzner` delegation instead of the legacy SSH server flow
2. the user-facing workflow is coherent for both direct and queue delegation
3. examples and command sequences match the current `ray-hetzner` repository behavior
4. sync, cleanup, and failure semantics are explicit and internally consistent

## Implementation Shape

The first implementation should focus on rewriting the existing skill and its mirrored documentation in this repository. It should not create a second skill or preserve the stale server-lifecycle text.
