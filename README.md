# hetzner-delegation

Mirror repository for the `hetzner-delegation` skill.

The skill now delegates work through `ray-hetzner` rather than the legacy one-server Hetzner SSH lifecycle.

Primary paths:

- direct Ray-cluster delegation for arbitrary project runs
- queue-backed delegation for `metaopt` batch workflows

Design and planning docs live under `docs/superpowers/`.
