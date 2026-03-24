---
name: forge
description: Write smart contracts from canonical patterns
triggers:
  - /forge
  - write contract
  - build contract
  - contract architecture
capabilities:
  model_tier: opus
  danger_level: high
  effort_hint: large
  downgrade_allowed: false
---

# Forge — Contract Writing from Canon

Write smart contracts grounded in the NOETHER canon — registry patterns, role-gated access, launch lifecycle, bitmap tracking, and production-tested architectures.

## Workflow

1. Read the requirement
2. Check `knowledge/canon/` for prior art in the same domain
3. Check `knowledge/patterns/` for portable patterns that apply
4. Write the contract following NOETHER principles:
   - The contract is a vessel, not a brain
   - Roles are capabilities (use registry, not manual mappings)
   - Derive, don't store (element is math)
   - Solady over OpenZeppelin where gas matters
5. Validate deploy order with `/ceremony` when ready

## Pattern Library

| Category | Key Patterns |
|----------|-------------|
| Access Control | Registry hub-and-spoke, role-gated ops, capability-not-identity |
| Token | LibBitmap tracking, staged URI reveal, totalSupply counter |
| Lifecycle | Pause → configure → launch, unassigned-until-activated roles |
| Pricing | CDA (continuous decay + refund), DDA (step windows) |
| Distribution | Basis-point fee split, WAD-based shares |
| Upgrade | ERC-7201 namespaced storage, Beacon proxy, UUPS |
| Gas | Solady over OZ, derived-not-stored, bitmap-not-mapping |

## Output

Solidity contracts with:
- NatSpec documentation
- Role-based access via registry pattern
- Gas-optimized implementations
- Deploy ceremony notes
