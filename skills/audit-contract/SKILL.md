---
name: audit-contract
description: Review contracts against spec and canonical patterns
triggers:
  - audit contract
  - review contract
  - contract security
  - check contract
capabilities:
  model_tier: opus
  danger_level: moderate
  effort_hint: medium
  downgrade_allowed: false
---

# Audit Contract — Pattern-Grounded Review

Review smart contracts against the NOETHER canon. Find violations of principles, security concerns, pattern drift from established architectures, and anti-pattern usage.

## Review Dimensions

1. **Principle compliance** — does the contract follow NOETHER's 5 principles?
2. **Pattern alignment** — does it use canonical patterns or reinvent?
3. **Anti-pattern detection** — ERC721Enumerable, manual auth mappings, stored derivables?
4. **Security** — reentrancy, access control, overflow, frontrunning
5. **Gas efficiency** — Solady vs OZ choices, storage layout, bitmap usage
6. **Deploy readiness** — can this be deployed via ceremony?

## Output

Structured audit report with severity-classified findings.
