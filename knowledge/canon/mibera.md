---
codebase: mibera-contracts
author: team
chain: Berachain
solidity: "0.8.23-0.8.26"
repo: "0xHoneyJar/mibera-contracts"
---

# Mibera Contracts

Two sub-projects: a core NFT collection with fractured soulbound derivatives, and a full marketplace (honey-road) with lending, trading, vending, and revenue splitting.

## Core NFT

### ERC721C (Limit Break Creator Token Standards)

```solidity
import "creator-token-standards/ERC721C.sol";
```

ERC721C enforces creator-defined transfer policies on-chain. Royalties are not optional suggestions -- they're enforced at the contract level. Used for the primary Mibera collection.

### FracturedMibera (Soulbound Derivative)

Mirrors the parent Mibera collection. Each FracturedMibera token maps 1:1 to a parent token. Paid mint (not free claim). Soulbound -- cannot be transferred after minting.

```solidity
function _update(address from, address to, uint256 tokenId) internal override returns (address) {
    if (from != address(0)) revert Soulbound();
    return super._update(from, to, tokenId);
}

// Also blocks approvals
function approve(address, uint256) public pure override { revert Soulbound(); }
function setApprovalForAll(address, bool) public pure override { revert Soulbound(); }
```

This creates a two-token model: the tradeable Mibera NFT and the soulbound FracturedMibera that represents permanent membership or participation. Blocking approvals in addition to transfers prevents any route to moving the token.

## Honey-Road Marketplace

### Treasury with RFV + Two-Sided NFT-Collateralized Lending

Reserve Floor Value (RFV) backs each NFT with a calculable minimum value from the treasury.

```solidity
function realFloorValue() public view returns (uint256) {
    return (backing + backingLoanedOut - collateralHeld) / float;
    // where float = totalSupply - treasuryOwned
}
```

Two loan types: (a) borrow ETH against NFT collateral, (b) borrow NFTs against ETH collateral. Linear interest. Term limits. Default burns collateral. The treasury is the lender in both directions.

### CandiesMarket "Seized" Mechanic

```solidity
if (uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % 1000 == 0) {
    // 1-in-1000 chance: buyer gets SEIZED_ID (69420) instead of purchased item
    candies.mint(msg.sender, SEIZED_ID, totalAmount);
    return; // supply counters NOT incremented
}
```

One in 1000 purchases results in a "seized" variant -- an alternate token that replaces the expected one. The SEIZED_ID (69420) ties into the project's cultural motif. This creates emergent rarity without explicit rarity tiers. Supply counters are not incremented, so the seized token exists outside the normal supply accounting.

### P2P Trading System

Two trading contracts for different token standards:

- **CandiesTrade**: ERC1155 peer-to-peer trades
- **MiberaTrade**: ERC721 peer-to-peer trades

Both share the same pattern:

```solidity
struct TradeProposal {
    uint128 requestedTokenId;
    uint128 startedAt;     // = 1 storage slot
}
```

15-minute TTL on all trades. After expiry, trades are automatically invalid. Post-swap verification re-reads all balances after executing both transfers to catch non-standard token implementations:

```solidity
// After executing both transfers:
uint256 newBalance0 = candies.balanceOf(proposer, offeredId);
uint256 newBalance1 = candies.balanceOf(accepter, requestedId);
require(newBalance0 == expectedBalance0);
require(newBalance1 == expectedBalance1);
```

No long-lived approvals sitting in state. Targeted trades only (specific counterparty).

### VendingMachine Trait Uniqueness

```solidity
mapping(bytes32 => bool) private _usedTraitHashes;

function _validateTraits(uint256[] calldata traits) internal {
    bytes32 hash = keccak256(abi.encodePacked(traits));
    require(!_usedTraitHashes[hash], "Trait combination exists");
    _usedTraitHashes[hash] = true;
}
```

Hash-based deduplication ensures no two tokens share the same trait combination. O(1) per check. Always sort traits before hashing to ensure deterministic results.

### Splitter Revenue Distribution

Revenue splitting contract that handles both ERC20 and native BERA. WBERA is auto-unwrapped before distribution so recipients always receive native BERA.

```solidity
function distributeETH() external {
    WBERA.withdraw(WBERA.balanceOf(address(this))); // unwrap
    _distribute(address(this).balance);              // split by shares
}
```

Berachain-specific: WBERA at `0x6969...6969`. Recipients never need to unwrap manually.

### UUPS Across All Marketplace Contracts

Every contract in honey-road is UUPS upgradeable. Consistent upgrade pattern across the entire marketplace suite. Storage gaps in every base.

### Struct Packing

```solidity
struct Candy {
    uint128 price;
    uint64 currentSupply;
    uint64 maxSupply;
}
// price (128) + currentSupply (64) + maxSupply (64) = 256 bits = 1 storage slot
```

Deliberate struct sizing to fit exactly one EVM storage slot (32 bytes / 256 bits). This halves SLOAD/SSTORE costs compared to a naive struct that spans two slots.

## The "42" Motif

The number 42 appears throughout the codebase as a design signature:

- 42% maximum discount cap
- 42 bps per-NFT discount
- 4.20% interest rate on loans
- 42 WETH as a `noFloat` constant
- 4.2-month loan term
- SEIZED_ID: 69420

This is intentional thematic consistency, not coincidence. When adding new constants to this codebase, consider whether 42-derived values are appropriate.

## Key Takeaways for Future Agents

1. **ERC721C** for royalty enforcement -- not EIP-2981 (which is advisory). Use Limit Break's creator-token-standards.
2. **Soulbound derivatives** (FracturedMibera pattern) create permanent membership tokens alongside tradeable assets. Block both transfers AND approvals.
3. **1-in-N random mechanics** (seized) create emergent rarity. Use a sentinel value that ties to project lore.
4. **15-min TTL trades** with post-swap verification prevent stale orders and ensure atomic execution.
5. **Hash dedup for trait uniqueness** is O(1) per check. Always sort traits before hashing.
6. **Struct packing to 256 bits** is worth the effort. `uint128 + uint64 + uint64 = 1 slot`.
7. **Auto-unwrap WBERA** in splitters so recipients get native tokens. Don't force them to unwrap.
8. **Thematic constants** (42 motif) create codebase personality. Document them so future developers don't "fix" them.
