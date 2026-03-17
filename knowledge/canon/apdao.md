---
codebase: apdao-contracts
author: team
chain: Berachain bArtio
solidity: "0.8.23"
repo: "0xHoneyJar/apdao-contracts"
---

# apDAO — Canon Entry

## What It Is

Full DAO financial system: Nouns-style continuous auction with member sell queue, liquid backing treasury with RFV, NFT-collateralized loans, governance token coupling, multi-collection deposit for seat claiming.

## Key Patterns

### Nouns-Style Auction with Member Sell Queue

```solidity
function _createAuction() internal {
    if (auctionQueue.length > 1) {
        // VRF random selection from queue
        uint256 index = randomNumber % auctionQueue.length;
        tokenId = auctionQueue[index];
    } else {
        // Mint new token
        tokenId = apiologyToken.mint();
    }
}
```

Members can `addToAuctionQueue(tokenIds)` to list NFTs for sale. Auction house takes custody. VRF selects which queued token goes next. Dual source: mint new OR sell existing.

### Dynamic Reserve Price from Treasury RFV

```solidity
function _computeReservePrice() internal view returns (uint256) {
    uint256 rfv = treasury.realFloorValue();
    return rfv + (rfv * reservePriceBuffer / 100);
}
```

Auction can't sell below backing value. Buffer adds safety margin. Treasury RFV is dynamic — changes with deposits, fees, loans.

### Dual VRF Support

```solidity
// Toggle between Pyth Entropy and Chainsight VRF
if (usePythEntropy) {
    IEntropy(entropy).requestWithCallback{value: fee}(provider, userRandom);
} else {
    IChainsightVRF(chainsightVRF).requestRandomWords(...);
}
```

Two VRF providers in one contract, toggle-able. Operational flexibility — switch providers without redeployment.

### Soulbound-ish Transfers with Governance Coupling

```solidity
function _update(address from, address to, uint256 tokenId) internal override {
    // Only allow transfers to/from LBT or Auction House
    if (from != address(0) && to != address(0)) {
        if (!isLBTOrAuctionHouse(from) && !isLBTOrAuctionHouse(to)) {
            if (!authorizedTransferors[from]) revert TransferRestricted();
        }
    }
    // Governance token coupling: 1 NFT = 1 governance token
    if (from != address(0)) stationXToken.burn(from, 1 ether);
    if (to != address(0)) stationXToken.mint(to, 1 ether);
}
```

NFTs tied to governance participation. Every mint/transfer adjusts governance tokens. Open market trading blocked (only through auction or treasury).

### No-Bid Settlement

```solidity
if (lastBidder == address(0)) {
    if (isQueuedToken) {
        // Queued: redeem at RFV, return to original owner
        treasury.redeemItem(tokenId);
    } else {
        // Minted: just burn it
        apiologyToken.burn(tokenId);
    }
}
```

Guarantees sellers receive at least redemption value. Minted tokens that get no bids are simply burned — no dead inventory.

### NFT-Collateralized Loans

```solidity
function receiveLoan(uint256[] tokenIds) external {
    uint256 loanAmount = tokenIds.length * realFloorValue() * (10000 - ROYALTY_PERCENT) / 10000;
    // Transfer NFTs as collateral
    // Send WETH to borrower
    // Track loan with interest
}
```

Deposit NFTs, borrow up to (100 - royalty)% of their backing. Linear interest. Term limit. Expired loans → collateral burned, backing adjusted.
