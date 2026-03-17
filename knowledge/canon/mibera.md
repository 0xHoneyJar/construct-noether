---
codebase: mibera-contracts
author: team
chain: Berachain
solidity: "0.8.23-0.8.26"
repo: "0xHoneyJar/mibera-contracts"
---

# Mibera — Canon Entry

## What It Is

Two sub-projects: core NFT collection (10k ERC-721 + soulbound derivative + treasury) and honey-road marketplace (ERC-1155 items + P2P trading + vending machine + revenue splitter). 12 contracts total.

## Key Patterns

### ERC721C (Creator Token Standards)

```solidity
contract Mibera is ERC721C, ...
```

Limit Break's on-chain royalty enforcement. Goes beyond ERC-2981 advisory — enforces transfer restrictions at contract level. Used for the core 10k collection.

### Soulbound Derivative (FracturedMibera)

```solidity
function _update(address from, address to, uint256 tokenId) internal override returns (address) {
    if (from != address(0)) revert Soulbound();
    return super._update(from, to, tokenId);
}

// Also blocks approvals
function approve(address, uint256) public pure override { revert Soulbound(); }
function setApprovalForAll(address, bool) public pure override { revert Soulbound(); }
```

Mirrors parent collection — same tokenIds. Paid soulbound mint. Blocks transfers AND approvals.

### Treasury with RFV + Two-Sided Lending

```solidity
function realFloorValue() public view returns (uint256) {
    return (backing + backingLoanedOut - collateralHeld) / float;
    // where float = totalSupply - treasuryOwned
}
```

Two loan types: (a) borrow ETH against NFT collateral, (b) borrow NFTs against ETH collateral. Linear interest. Term limits. Default burns collateral.

### P2P Trading with Post-Swap Verification

```solidity
// After executing both transfers:
uint256 newBalance0 = candies.balanceOf(proposer, offeredId);
uint256 newBalance1 = candies.balanceOf(accepter, requestedId);
require(newBalance0 == expectedBalance0);
require(newBalance1 == expectedBalance1);
```

Targeted trades (specific counterparty). 15-minute TTL. Re-reads all balances after swap to catch non-standard token implementations.

### "Seized" Randomness Mechanic

```solidity
if (uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % 1000 == 0) {
    // 1-in-1000 chance: buyer gets SEIZED_ID (69420) instead of purchased item
    candies.mint(msg.sender, SEIZED_ID, totalAmount);
    return; // supply counters NOT incremented
}
```

Gamification at the contract level. Buyer pays full price but gets a surprise token. Fun.

### Struct Packing

```solidity
struct Candy { uint128 price; uint64 currentSupply; uint64 maxSupply; } // = 1 slot
struct TradeProposal { uint128 requestedTokenId; uint128 startedAt; } // = 1 slot
```

### Revenue Splitter with WBERA Auto-Unwrap

```solidity
function distributeETH() external {
    WBERA.withdraw(WBERA.balanceOf(address(this))); // unwrap
    _distribute(address(this).balance); // split by shares
}
```

Berachain-specific: WBERA at `0x6969...6969`.

### The "42" Motif

Max discount: 42%. Per-NFT discount: 42 bps. Interest: 4.20%. NoFloat: 42 WETH. Term: 4.2 months. SEIZED_ID: 69420. Cultural numbers throughout.
