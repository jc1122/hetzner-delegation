# hetzner-delegation

This project contains the `hetzner-delegation` Claude Code skill, developed to delegate computational work to Hetzner Cloud servers.

The actual skill lives at `~/.claude/skills/hetzner-delegation/SKILL.md` — this repo mirrors it for version control and sharing.

## What was built

### hetzner-delegation skill
- **Location:** `~/.claude/skills/hetzner-delegation/SKILL.md` (mirrored as `SKILL.md` here)
- **GitHub:** jc1122/hetzner-delegation
- **Purpose:** Skill for spawning/reusing Hetzner servers, delegating experiments, snapshotting, and deleting

Key design decisions made during development:
- Default server type: `cx53` (20 vCPU / 40 GB) for heavy compute
- Workflow: check idle servers first → provision from snapshot if none → run → snapshot → delete
- Local agent controls server over SSH — no remote agent spawned
- `trap cleanup EXIT` ensures snapshot+delete even on failure
- When dispatching subagents for compute work, the subagent prompt must explicitly say to use the hetzner-delegation skill

### ray-hetzner scaffolding
- **Location:** `~/projects/ray-hetzner/`
- **GitHub:** jc1122/ray-hetzner
- **Purpose:** Universal Ray cluster on Hetzner for running multiple projects in parallel

Architecture:
```
Laptop / agents  →  ray.init("ray://cx11-ip:10001")
                         ↓
                    cx11 (Ray head, persistent, ~€4/mo)
                         ↓
                    cx53 (ephemeral workers, spawned/killed per campaign)
```

Scripts:
- `setup_head.sh`    — provision cx11 head (idempotent)
- `add_worker.sh`    — spin up cx53, join Ray cluster
- `remove_worker.sh` — drain, snapshot, delete worker
- `teardown_head.sh` — snapshot, delete head
- `status.sh`        — hcloud + ray status
- `connect_test.py`  — verify laptop → cluster connectivity

Config lives in `~/projects/ray-hetzner/config.env` (gitignored — contains HCLOUD_TOKEN).

## Current state

The hetzner-delegation skill is complete and published. The ray-hetzner scaffolding is created but **not yet configured or tested** — the cx11 head node has not been provisioned yet.

## Next steps

1. Fill in `~/projects/ray-hetzner/config.env` (HCLOUD_TOKEN, HETZNER_SSH_KEY)
2. Run `./setup_head.sh` to provision the cx11 head node
3. Add a cx53 worker with `./add_worker.sh`
4. Verify with `python connect_test.py`
5. Integrate Ray into individual projects (e.g. MarketNN's `run_edge_size_baseline.py`)

## Relationship to other projects

- **MarketNN** (`~/projects/MarketNN/`) — first candidate for Ray integration; has a 36-combination regression grid (6 instruments × 3 label modes × 2 model types) that is embarrassingly parallel
- **ray-hetzner** (`~/projects/ray-hetzner/`) — the cluster infrastructure; independent of any single project

## Keeping skill in sync

After editing `~/.claude/skills/hetzner-delegation/SKILL.md`, sync to this repo:
```bash
cp ~/.claude/skills/hetzner-delegation/SKILL.md ~/projects/hetzner-delegation/SKILL.md
cd ~/projects/hetzner-delegation
git add SKILL.md && git commit -m "..." && git push
```
