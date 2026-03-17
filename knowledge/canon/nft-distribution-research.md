---
codebase: nft-distribution-research
author: exp
solidity: "0.8.x"
repo: "exp-table/nft-distribution-research"
---

# NFT Distribution Research — Canon Entry

## What It Is

Focused PoC exploring two Dutch Auction pricing mechanisms for NFT mints. Not production code — pricing research.

## Key Patterns

### Continuous Dutch Auction (CDA)

```solidity
struct Auction {
    uint128 startingPrice;       // slot 1 upper
    uint64  decreasingConstant;  // slot 1 lower
    uint32  startingBlock;       // slot 2 upper
    uint32  period;              // slot 2 lower
}
// Total: 128+64+32+32 = 256 bits = exactly 1 slot
```

Price decays linearly per block: `price = startingPrice - (decreasingConstant * blocksElapsed)`. Floor at `period` blocks.

**Refund pattern** (critical for UX):
```solidity
if (msg.value - pricePaid > 0) {
    Address.sendValue(payable(msg.sender), msg.value - pricePaid);
}
```
Buyer sends "enough," gets change. No exact-wei calculation needed.

### Discrete Dutch Auction (DDA)

Price drops in step-function intervals. Stable for `interval` blocks at each tier.

```
Blocks 0-49:   5 ETH
Blocks 50-99:  4.5 ETH
Blocks 100+:   4 ETH (floor)
```

Calm pricing windows. Buyers have time to decide at a known price. No gas war.

**DDA uses exact match** (no refund): `require(msg.value == price)`. Appropriate when price is stable and predictable within the window.

### Struct Packing for Gas

CDA's struct is optimal: 128+64+32+32 = 256 bits = 1 storage slot.
DDA's struct is naive: `uint256 startingPrice` wastes a full slot alone.

**Lesson**: Always pack auction/config structs to slot boundaries.

### Block-Based Timing

Both use `block.number`, not `block.timestamp`. Timestamps can be manipulated by validators within ~12s. Block numbers cannot. For predictable auction timing, use blocks.

### Abstract Base + Mock Pattern

Both auctions are `abstract`. Concrete contracts inherit and add `buy()`. The auction logic is completely decoupled from token logic. `verifyBid(auctionId)` is `internal` — the parent calls it before minting.

## Recommendation for NFT Projects

**DDA + CDA refund hybrid**:
- DDA's stable price windows for calm, unhurried minting
- CDA's refund pattern for soft UX (buyer overpays, gets change)
- CDA's struct packing for gas efficiency
- Block-based timing for manipulation resistance
