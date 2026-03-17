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

### Canon — 13 Codebases Analyzed

| Codebase | Author | Key Patterns |
|----------|--------|-------------|
| **Crayons** | exp | Beacon proxy, ERC-7201 namespaced storage, UUPS Controller, launch lifecycle |
| **HoneyJar** | team | Registry hub-and-spoke, LibBitmap, role-gated burn, MultisigOwnable |
| **Dove Protocol** | exp | Bit-packed codec (assembly), dual-bridge, index-based fees, voucher IOUs |
| **NFT Distribution Research** | exp | CDA/DDA Dutch auctions, struct packing, refund patterns |
| **Badges** | exp | Soulbound `_update`, per-badge Merkle roots, idempotent batch, NFT-gated pricing |
| **Faucet (Hive)** | team | Multi-round Merkle distributor, Solady-only (76 lines) |
| **HenloFlip** | team | Pyth Entropy VRF, progressive jackpot, event-only factory |
| **Mibera** | team | ERC721C royalties, P2P trading, treasury RFV, seized mechanic, semantic struct packing |
| **fatBERA** | team | ERC-4626 LST, Synthetix rewards, whitelisted vault composability |
| **apDAO** | team | Nouns auction + sell queue, dual VRF, NFT lending, governance coupling |
| **Interpol** | team | Double Beacon Proxy, adapter/strategy pattern, badge fee discounts |
| **Henlo V2** | team | Semantic token IDs (HENLOCKED), oracle-gated redemption, dual-verification airdrop |
| **SF Contracts V2** | team | Profit unlock anti-sandwich, module composition, HENLO rebate currency |

### Principles
1. The contract is a vessel, not a brain
2. Roles are capabilities
3. Element is math, not storage
4. The deploy is a ceremony
5. Overpay and refund

## Pattern Library

Patterns extracted from the canon, organized by category:

| Category | Patterns |
|----------|---------|
| **Access Control** | Registry hub-and-spoke, role-gated operations, operator delegation, badge-based discounts |
| **Token Patterns** | LibBitmap tracking, staged URI reveal, soulbound `_update`, ERC721C royalties, semantic token IDs |
| **Lifecycle** | Pause → configure → launch, unassigned-until-activated roles, paused-on-deploy with whitelist |
| **Pricing** | CDA refund, DDA step windows, NFT-gated tiers, dynamic reserve from RFV |
| **Distribution** | Multi-round Merkle (bitmap per root), selective mint from entitlement, idempotent batch, dual-verification |
| **DeFi** | ERC-4626 vaults, Synthetix reward streaming, profit unlock, whitelisted vault composability |
| **Randomness** | Pyth Entropy (callback), Chainlink VRF, Chainsight VRF, prevrandao (testnet), dual-modulus |
| **Trading** | P2P targeted trades, time-bounded proposals, post-swap verification, Nouns auction + sell queue |
| **Upgrade** | ERC-7201 storage, Beacon proxy, UUPS, storage gaps, transparent proxy |
| **Gas** | Solady over OZ, derived-not-stored, bitmap-not-mapping, struct packing to slot boundaries |
| **Cross-chain** | Bit-packed codec, dual-bridge (message + asset), trusted remote whitelisting |
| **Gamification** | Progressive jackpot, seized mechanic, event-only factory, game pot cap |

## Composes With

- **Protocol** — NOETHER architects, Protocol verifies. Run `/forge` then `/verify`.

## License

MIT
