---
codebase: henlo-contracts-V2
author: team
chain: Berachain
solidity: "0.8.23"
repo: "0xHoneyJar/henlo-contracts-V2"
---

# Henlo V2 — Canon Entry

## What It Is

HENLO ERC-20 token + oHENLO options + HENLOCKED strike-price system + soulbound game collection + dual-verification airdrop.

## Key Patterns

### Semantic Token IDs (HENLOCKED)

```solidity
// ERC-1155 where token ID IS the strike price
mapping(uint256 strike => bool) public strikeAvailable;

function name(uint256 strike) external view returns (string memory) {
    // Converts to human-readable: "henlocked@420m", "henlocked@4.2b"
}
```

The token ID itself carries data. No additional mapping needed. Dynamic name generation from the ID.

### Oracle-Gated Redemption

```solidity
function canRedeem(uint256 strike) public view returns (bool) {
    (, int256 price,, uint256 updatedAt,) = oracle.latestRoundData();
    if (block.timestamp - updatedAt > 1 hours) return false; // staleness
    return uint256(price) >= strike;
}
```

Positions unlock when oracle price meets strike. 1-hour staleness check.

### Non-Transferable Soulbound ERC-1155

```solidity
function _beforeTokenTransfer(address, address from, address, uint256[] memory, uint256[] memory, bytes memory) internal override {
    if (from != address(0)) revert("Soulbound");
}
```

Three token types: Luminary (0), Renegade (1), Drifter (2). Only mintable, never transferable.

### Dual-Verification Airdrop

```solidity
function claim(uint256 index, uint256 amount, bytes32[] proof, bytes signature) external {
    // 1. Verify Merkle proof
    require(MerkleProofLib.verify(proof, root, leaf));
    // 2. Verify ECDSA signature
    require(ECDSA.recover(hash, signature) == signer);
    // Both must pass
}
```

Defense-in-depth. Compromised Merkle tree alone can't drain. Compromised signer alone can't drain.

### Option Token (oHENLO)

```solidity
function exercise(uint256 amount) external {
    uint256 cost = amount.mulWad(exercisePrice);
    // Pay ETH, receive HENLO, oHENLO burned
}
```

ERC-20 option with block-based expiry. Transfer-restricted (whitelist only). Burns on exercise.

### Paused-on-Deploy with Transfer Whitelist

```solidity
constructor() {
    _pause();
    _grantRole(TRANSFER_WHITELIST, address(0)); // allow minting while paused
}
```

Deploy paused. Selectively whitelist addresses for controlled distribution before global unpause.
