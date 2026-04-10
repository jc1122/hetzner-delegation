# hetzner-delegation

This repository mirrors the `hetzner-delegation` skill for version control.

Current direction:

- this repository is the versioned source for the agent-facing skill
- `~/projects/ray-hetzner` is the execution backend and runtime source of truth
- Aorus is the permanent Ray head and queue control plane
- queue mode is the primary execution path
- direct mode uses `submit_ray_job.sh` for one-off Ray jobs
- the old manual Hetzner head and worker lifecycle is removed
- large data movement must stay explicit

If another runtime still uses a copied live skill, sync this repository's `SKILL.md` out after editing:

```bash
cd ~/projects/hetzner-delegation
cp SKILL.md ~/.claude/skills/hetzner-delegation/SKILL.md
```

See `docs/superpowers/specs/` for historical design notes.
