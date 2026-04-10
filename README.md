# hetzner-delegation

Mirror repository for the agent-facing `hetzner-delegation` skill.

This repo keeps the skill contract in version control. The actual execution backend lives in [`ray-hetzner`](https://github.com/jc1122/ray-hetzner).

Current model:

- Aorus is the permanent Ray head and queue control plane
- Hetzner nodes are autoscaled disposable worker capacity
- queue-backed `metaopt` execution is the primary path
- direct one-off Ray jobs are the secondary path

The old manual head and worker lifecycle has been removed from the backend. The supported surface is the Aorus bootstrap path, queue thin-client path, direct Ray Jobs helper, autoscaler configuration, and supporting queue/runtime code.

The skill definition is in [SKILL.md](SKILL.md). For the actual runtime behavior, start from the [`ray-hetzner` README](https://github.com/jc1122/ray-hetzner/blob/master/README.md) and `docs/OPERATIONS.md` in that repository.

Design specs live under `docs/superpowers/specs/`.
