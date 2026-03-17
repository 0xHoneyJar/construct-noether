# ⚒️ NOETHER

> every symmetry has a conservation law. every role has a capability.

Smart contract architecture and development construct. Named after Emmy Noether — mathematician who proved that symmetries correspond to conservation laws.

**Protocol asks: "does the frontend match the chain?"**
**NOETHER asks: "how should we build this contract?"**

## Install

```bash
/constructs install noether
```

## What's Inside

### Skills
- **forge** — Write contracts from canonical patterns
- **audit-contract** — Review contracts against spec, find violations and drift
- **ceremony** — Validate deploy sequences (hub before spokes, roles before activation)
- **excavate** — Study existing codebases and extract patterns into the canon
- **evolve** — Plan contract upgrades (what's conserved, what changes)

### Canon (Knowledge Base)
Extracted patterns from production codebases:
- **Crayons** (exp) — Beacon proxy, ERC-7201, UUPS Controller, launch lifecycle
- **HoneyJar** (team) — Registry hub-and-spoke, LibBitmap, role-gated burn, MultisigOwnable
- **Dove Protocol** (exp) — Bit-packed codec, dual-bridge, index-based fees, voucher IOUs
- **NFT Distribution Research** (exp) — CDA/DDA Dutch auctions, struct packing

### Principles
1. The contract is a vessel, not a brain
2. Roles are capabilities
3. Element is math, not storage
4. The deploy is a ceremony
5. Overpay and refund

## Composes With

- **Protocol** — NOETHER architects, Protocol verifies. Run `/forge` then `/verify`.

## License

MIT
