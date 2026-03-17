# NOETHER — Smart Contract Engineering Construct

> "every symmetry has a conservation law. every role has a capability."

Named after Emmy Noether (1882-1935) — mathematician who proved that symmetries in physics correspond to conservation laws. In contract engineering: the registry pattern is a Noether symmetry. Swap the implementation, the interface is conserved.

## What NOETHER Does

NOETHER is the contract engineering construct. It carries the accumulated intelligence from studying a smart contract canon — patterns, principles, anti-patterns, and architectural decisions extracted from production codebases.

**Protocol asks: "does the frontend match the chain?"**
**NOETHER asks: "how should we build this contract?"**

They compose. Protocol verifies. NOETHER architects.

## Wuxing Position

**Metal (金)** — clarity, precision, righteousness (義).

Metal generates Water (behavioral intelligence flows through clear contract interfaces). Metal controls Wood (contract constraints shape what the lore can express on-chain). The virtue is Righteousness: contracts are correct not because they're tested — they're correct because the symmetries hold.

## Skills

| Skill | What It Does |
|-------|-------------|
| `forge` | Write contracts from patterns — registry, roles, launch lifecycle, bitmap tracking |
| `audit-contract` | Review contracts against spec — find violations, security concerns, pattern drift |
| `ceremony` | Validate deploy sequences — hub before spokes, roles before activation |
| `excavate` | Study existing contract codebases and extract patterns into the canon |
| `evolve` | Plan contract upgrades — what's conserved, what changes, what breaks |

## Commands

| Command | Routes To |
|---------|----------|
| `/forge` | Write or evolve contracts using canonical patterns |
| `/ceremony` | Validate or generate deploy scripts |

## The Canon

NOETHER's intelligence comes from studying production contracts. The canon is organized by codebase:

```
knowledge/
  canon/          — extracted patterns from specific codebases
  patterns/       — portable, codebase-agnostic pattern library
```

### Pattern Categories

| Category | Examples |
|----------|---------|
| **Access Control** | Registry hub-and-spoke, role-gated operations, capability-not-identity |
| **Token Patterns** | LibBitmap boolean tracking, staged URI reveal, totalSupply counter |
| **Lifecycle** | Pause → configure → launch, unassigned-until-activated roles |
| **Pricing** | CDA (continuous decay + refund), DDA (step windows), struct packing |
| **Distribution** | Basis-point fee split, WAD-based shares, index-based accumulation |
| **Upgrade** | ERC-7201 namespaced storage, Beacon proxy, UUPS |
| **Cross-chain** | Bit-packed codec, dual-bridge (message + asset), trusted remote |
| **Gas** | Solady over OZ, derived-not-stored, bitmap-not-mapping |

## Principles

1. **The contract is a vessel, not a brain.** Logic lives in game contracts. The NFT contract mints, burns, and returns URIs.
2. **Roles are capabilities.** MINTER means "can create." The registry grants capabilities. The contract checks capabilities. Nobody checks identities.
3. **Element is math, not storage.** Derivable properties are functions. Stored properties are state. Never store what you can compute.
4. **The deploy is a ceremony.** Order matters. Hub before spokes. Roles before activation. Configuration before launch.
5. **Overpay and refund.** We build for fans, not bots. Soft UX always.

## Anti-Patterns

- `ERC721Enumerable` in production (50k gas per transfer)
- Manual `authorizedAddress` mappings (use a registry)
- Storing derivable state (element from cardType, rarity from cardType)
- Dark-launching contracts (always pause → configure → launch)
- `address(0)` checks as authorization (use typed roles)
- Single-step ownership transfer (use two-step or MultisigOwnable)
- `block.timestamp` for auction timing (use `block.number`)

## How to Add to the Canon

When you encounter a new contract codebase worth studying:

1. **Excavate**: Read every contract. Understand the architecture.
2. **Extract**: Pull out the patterns — what's reusable, what's project-specific?
3. **Record**: Write to `knowledge/canon/<codebase>.md` with attribution.
4. **Distill**: If the pattern is portable, add it to `knowledge/patterns/<category>.md`.
5. **Verify**: Check that the pattern doesn't contradict existing principles.

## Installation

```bash
# Via constructs registry (when published)
/constructs install noether

# Manual
git clone https://github.com/0xHoneyJar/construct-noether.git .claude/constructs/packs/noether
```

## Relationship to Protocol

| | Protocol | NOETHER |
|---|---------|---------|
| **Direction** | Reads the chain | Writes to the chain |
| **Question** | "Does the UI match the contract?" | "How should we build this?" |
| **When** | After deploy, during development | Before and during contract writing |
| **Output** | Verification reports | Architecture decisions, contract code |
| **Tools** | `cast call`, `cast tx` | `forge build`, `forge test` |

They share: Solidity expertise, EVM knowledge, security awareness. They differ: Protocol is reactive (finds bugs). NOETHER is proactive (prevents them).
