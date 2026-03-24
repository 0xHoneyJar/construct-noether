---
name: ceremony
description: Validate deploy sequences and generate deploy scripts
triggers:
  - /ceremony
  - deploy ceremony
  - deploy order
  - validate deploy
capabilities:
  model_tier: sonnet
  danger_level: high
  effort_hint: medium
  downgrade_allowed: false
---

# Ceremony — Deploy Sequence Validation

Validate and generate deploy scripts that respect the canonical order: hub before spokes, roles before activation, configuration before launch.

## Principles

- Deploy order is not optional — it's a ceremony
- Hub contracts deploy first (registry, router)
- Spoke contracts register with hubs after deploy
- Roles are granted after all contracts are deployed
- Configuration is set before unpausing
- Never dark-launch — always pause → configure → launch

## Output

Ordered deploy script with verification checkpoints.
