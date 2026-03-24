---
name: evolve
description: Plan contract upgrades — what's conserved, what changes, what breaks
triggers:
  - evolve contract
  - upgrade contract
  - contract migration
  - plan upgrade
capabilities:
  model_tier: opus
  danger_level: high
  effort_hint: large
  downgrade_allowed: false
---

# Evolve — Contract Upgrade Planning

Plan contract upgrades through the lens of Noether's theorem: identify what's conserved (the symmetries), what changes (the transformations), and what breaks (the violations).

## Analysis Framework

1. **Conservation** — what interfaces, storage slots, and roles must remain?
2. **Transformation** — what logic, patterns, or implementations change?
3. **Violation** — what breaks if we get the migration wrong?
4. **Ceremony** — what's the upgrade deploy sequence?

## Upgrade Patterns

- ERC-7201 namespaced storage (safe slot management)
- Beacon proxy (coordinated upgrades across clones)
- UUPS (self-upgrade with access control)
- Diamond (selective facet replacement)

## Output

Upgrade plan with conservation analysis, migration steps, and rollback strategy.
