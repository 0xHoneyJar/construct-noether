---
name: excavate
description: Study contract codebases and extract patterns into the canon
triggers:
  - excavate
  - study contracts
  - extract patterns
  - read contracts
capabilities:
  model_tier: opus
  danger_level: safe
  effort_hint: large
  downgrade_allowed: true
---

# Excavate — Canon Extraction

Study existing smart contract codebases. Extract architectural patterns, design decisions, and anti-patterns into the NOETHER canon.

## Process

1. Read every contract in the codebase
2. Map the architecture — what connects to what, who calls whom
3. Extract reusable patterns — access control, lifecycle, token, pricing
4. Record to `knowledge/canon/<codebase>.md` with attribution
5. Distill portable patterns to `knowledge/patterns/<category>.md`
6. Verify new patterns don't contradict existing principles

## Output

Canon entry with architecture map, extracted patterns, and design rationale.
