---
codebase: apdao-contracts
author: team
chain: Berachain bArtio
solidity: "0.8.23"
repo: "0xHoneyJar/apdao-contracts"
---

# apDAO Contracts

Nouns-style continuous auction DAO with NFT-collateralized lending and dual VRF support.

## Core Architecture

### Continuous Auction with Member Sell Queue

Inspired by Nouns DAO auction mechanics but with a twist: existing members can queue their NFTs for sale, and VRF randomly selects which queued NFT goes to auction next.

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

Members can `addToAuctionQueue(tokenIds)` to list NFTs for sale. Auction house takes custody. Dual source: mint new OR sell existing. VRF random selection ensures fair ordering -- no front-running the sell queue. Sellers don't know when their turn comes, which prevents manipulation.

### Dynamic Reserve Price from Treasury RFV

The auction reserve price is not fixed. It's computed from the treasury's Reserve Floor Value plus a safety buffer.

```solidity
function _computeReservePrice() internal view returns (uint256) {
    uint256 rfv = treasury.realFloorValue();
    return rfv + (rfv * reservePriceBuffer / 100);
}
```

As the treasury grows, the minimum bid increases. This creates a self-reinforcing floor -- the DAO's NFTs are always worth at least their proportional share of the treasury. Buffer adds safety margin. Treasury RFV is dynamic -- changes with deposits, fees, loans.

### NFT-Collateralized Loans

NFT holders can borrow against their tokens up to the per-token RFV. The treasury is the lender.

```solidity
function receiveLoan(uint256[] tokenIds) external {
    uint256 loanAmount = tokenIds.length * realFloorValue() * (10000 - ROYALTY_PERCENT) / 10000;
    // Transfer NFTs as collateral
    // Send WETH to borrower
    // Track loan with interest
}
```

Deposit NFTs, borrow up to (100 - royalty)% of their backing. Linear interest. Term limit. Expired loans result in collateral being burned and backing adjusted. Liquidation triggers if the loan value exceeds the collateral's current RFV (which can change as treasury value fluctuates).

### Soulbound-ish Transfers with Governance Token Coupling

Transfers are restricted but not fully soulbound. NFTs are coupled with StationX ERC20 governance tokens -- every mint/transfer adjusts governance tokens automatically.

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

Open market trading is blocked -- transfers only through auction or treasury. This ensures governance power stays coupled with NFT ownership. You can't sell the NFT and keep the votes.

### Dual VRF Support

Toggle-able between Pyth Entropy and Chainsight VRF. Both follow the request-callback pattern but with different provider interfaces.

```solidity
enum VRFProvider { PYTH_ENTROPY, CHAINSIGHT }
VRFProvider public activeProvider;

function requestRandomness() internal returns (uint64) {
    if (usePythEntropy) {
        return IEntropy(entropy).requestWithCallback{value: fee}(provider, userRandom);
    } else {
        return IChainsightVRF(chainsightVRF).requestRandomWords(...);
    }
}
```

Dual VRF provides resilience -- if one provider goes down or raises fees, the DAO can switch without redeployment. The toggle is governance-controlled. Abstract the VRF interface and make the provider toggle-able.

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

Guarantees sellers receive at least redemption value. Minted tokens that get no bids are simply burned -- no dead inventory accumulates.

### Multi-Collection NFT Deposit for DAO Seat Claiming

Users can deposit NFTs from multiple approved collections to claim a DAO seat. Not limited to the DAO's own NFT.

```solidity
mapping(address collection => bool) public approvedCollections;

function depositForSeat(address collection, uint256 tokenId) external {
    require(approvedCollections[collection], "Not approved");
    IERC721(collection).transferFrom(msg.sender, address(this), tokenId);
    _mintDAOSeat(msg.sender);
}
```

This allows the DAO to absorb members from partner collections, creating cross-collection governance.

### Transparent Proxy (Not UUPS)

The auction house uses OpenZeppelin's TransparentUpgradeableProxy, not UUPS. The admin is a separate ProxyAdmin contract.

```solidity
new TransparentUpgradeableProxy(
    implementationAddress,
    proxyAdminAddress,
    abi.encodeWithSignature("initialize(...)")
);
```

Transparent proxy was chosen over UUPS because the auction house is the most critical contract -- the extra gas cost per call is worth the stronger separation between admin and user code paths. The ProxyAdmin is controlled by the DAO multisig.

## Key Takeaways for Future Agents

1. **VRF-selected sell queue** prevents front-running in continuous auctions. Random ordering is fair ordering.
2. **Dynamic reserve from treasury RFV** creates a self-reinforcing price floor. The floor grows with the treasury.
3. **Coupled NFT + governance tokens** ensures voting power tracks ownership. Override `_update` to enforce coupling with mint/burn on transfer.
4. **Dual VRF providers** (Pyth + Chainsight) provide resilience. Abstract the VRF interface and make the provider toggle-able.
5. **No-bid settlement** differentiates queued vs minted tokens. Queued tokens redeem at RFV; minted tokens burn. Never accumulate dead inventory.
6. **Multi-collection deposits** for DAO seats enable cross-collection governance. Good for ecosystem DAOs.
7. **Transparent proxy** over UUPS when the contract is critical enough to justify the gas overhead for stronger admin/user separation.
