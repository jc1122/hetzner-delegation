# hetzner-delegation

Mirror repository for the agent-facing `hetzner-delegation` skill.

The skill delegates work through [`ray-hetzner`](https://github.com/jc1122/ray-hetzner) rather than the legacy one-server Hetzner SSH lifecycle. `hetzner-delegation` is the agent control layer; `ray-hetzner` is the execution backend it drives.

Primary paths:

- direct Ray-cluster delegation for arbitrary project runs
- queue-backed delegation for `metaopt` batch workflows

The skill definition is in [SKILL.md](SKILL.md). For the underlying cluster/runtime flow, start from the [`ray-hetzner` README](https://github.com/jc1122/ray-hetzner/blob/master/README.md).

Design specs live under `docs/superpowers/specs/`.
