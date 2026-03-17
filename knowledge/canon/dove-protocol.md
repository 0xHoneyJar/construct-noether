---
codebase: dove-protocol
author: exp
chain: L1↔L2 (Ethereum ↔ Polygon/Arbitrum)
solidity: "0.8.15"
repo: "dove-protocol/dove-protocol"
path: "src/"
---

# Dove Protocol — Canon Entry

## What It Is

dAMM (decentralized AMM): liquidity lives on L1, trading happens on L2. LPs deposit once on Ethereum, capital serves multiple L2 chains simultaneously.

## Key Patterns

### Bit-Packed Cross-Chain Codec (Assembly)

```solidity
// Codec.sol — hand-packed into 5 EVM words
fpacket := shl(253, SYNC_TO_L1)           // 3 bits msgType
fpacket := or(fpacket, shl(237, syncID))    // 16 bits
fpacket := or(fpacket, shl(223, syncerPercentage)) // 14 bits
fpacket := or(fpacket, shl(63, syncer))     // 160 bits address
```

Every bit position is documented. Minimizes relayer costs for cross-chain messages. Pattern: when payload size = cost, pack manually.

### Dual-Bridge Architecture

```
Hyperlane (fast, cheap) → metadata/coordination messages
Stargate/LayerZero (slower) → actual token transfers
```

Decoupled by design. Messages can arrive in any order. `syncID` correlates them. Resilience: even if one layer fails, the other is independent.

### Index-Based Fee Accumulation (Solidly/Velodrome)

```solidity
uint256 public index0;   // global cumulative fees per LP token
mapping(address => uint256) public supplyIndex0;
mapping(address => uint256) public claimable0;

function _updateFor(address recipient) internal {
    uint256 _supplied = balanceOf[recipient];
    if (_supplied > 0) {
        uint256 _delta = index0 - supplyIndex0[recipient];
        claimable0[recipient] += _supplied * _delta / 1e18;
    }
    supplyIndex0[recipient] = index0;
}
```

O(1) fee accrual. No iteration over holders. Settled lazily on transfer. Applied in `transfer()` and `transferFrom()` overrides.

### PVP Syncer Incentive

```solidity
function getSyncerPercentage() external view returns (uint64) {
    uint256 bps = ((block.timestamp - lastSyncTimestamp) * 6) / 100;
    return uint64(bps >= 5000 ? 5000 : bps);
}
```

~0.06 bps/second, caps at 50% after 24h. Creates an organic incentive without a dedicated relayer network. First to sync earns the percentage.

### Voucher IOUs

When L2 pair runs out of real tokens, it mints ERC20 vouchers (IOUs) instead of reverting. Capped at 10% of virtual balance. Redeemable on L2 (after sync) or L1 (burn + claim through Fountain).

**Pattern application**: staged NFT reveals. Mint a "voucher" card immediately, redeem for the real full-art card once metadata is settled. The cap maps to reveal batch sizing.

## When to Apply

- Cross-chain NFT metadata sync
- Any system where message cost depends on payload size (bit-pack)
- Pro-rata reward distribution without iteration (index pattern)
- Incentivizing permissionless keepers (syncer pattern)
