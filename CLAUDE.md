# hetzner-delegation

This repository mirrors the `hetzner-delegation` skill for version control.

Current direction:

- the old single-server Hetzner flow is obsolete
- the skill is rewritten around `~/projects/ray-hetzner`
- default behavior is delegation-first direct cluster execution
- queue mode is reserved for `metaopt`-shaped work
- automatic code sync is acceptable; large data sync must stay explicit

Sync the live skill back into this repository after editing:

```bash
cp ~/.claude/skills/hetzner-delegation/SKILL.md ~/projects/hetzner-delegation/SKILL.md
```
