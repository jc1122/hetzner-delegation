# hetzner-delegation

Mirror repository for the `hetzner-delegation` skill.

The skill now delegates work through `ray-hetzner` rather than the legacy one-server Hetzner SSH lifecycle.

Primary paths:

- direct Ray-cluster delegation for arbitrary project runs
- queue-backed delegation for `metaopt` batch workflows

The skill definition is in [SKILL.md](SKILL.md).

Design specs live under `docs/superpowers/specs/`.
