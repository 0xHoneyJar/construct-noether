---
codebase: faucet-contracts
author: team
chain: Berachain Artio
solidity: "0.8.23"
repo: "0xHoneyJar/faucet-contracts"
---

# Faucet Contracts

Minimal native ETH airdrop via multi-round Merkle distribution. 76 lines. The tightest contract in the codebase.

## Core Architecture

### Multi-Round Merkle Distributor

Same pattern as badges but for native ETH distribution.

```solidity
mapping(bytes32 => LibBitmap.Bitmap) private _claimedBitmaps;
```

Keyed by Merkle root. Each airdrop round publishes a new root. Old rounds remain claimable until the contract runs out of ETH.

### Solady-First Design

No OpenZeppelin at all. Pure Solady stack:
- `LibBitmap` for claim tracking
- `MerkleProofLib` for proof verification
- `Ownable` for access control

When you don't need upgradeability or complex access control, Solady alone is sufficient. This contract proves the minimal viable dependency set.

### Native ETH Distribution

```solidity
function claim(bytes32[] calldata proof, uint256 index, uint256 amount, bytes32 merkleRoot) external {
    // verify proof
    // mark claimed
    (bool sent,) = msg.sender.call{value: amount}("");
    require(sent, "Transfer failed");
}
```

Not ERC20. Raw ETH. The contract holds ETH and distributes it directly. `receive()` function accepts deposits.

### Dual Distribution Model

Two ways ETH leaves the contract:

1. **Merkle claims**: Users submit proofs to claim their allocation. Permissionless, trustless.
2. **Owner drip**: Owner can send ETH to arbitrary addresses. Used for manual top-ups, special allocations, or emergency distribution.

```solidity
function drip(address to, uint256 amount) external onlyOwner {
    (bool sent,) = to.call{value: amount}("");
    require(sent, "Transfer failed");
}
```

The drip function is the escape hatch. When Merkle trees are too slow or too rigid, the owner can distribute directly.

## Key Takeaways for Future Agents

1. **76 lines is achievable** for a Merkle distributor. If your airdrop contract is over 200 lines, you're overbuilding.
2. **Solady-only is valid** when you don't need upgradeability. Don't import OZ out of habit.
3. **Native ETH airdrops** are simpler than ERC20 -- no approve/transferFrom dance, no token contract dependency.
4. **Owner drip as escape hatch** alongside Merkle claims gives operational flexibility without compromising the trustless path.
5. **`mapping(bytes32 => LibBitmap.Bitmap)`** is the reusable primitive. Same pattern appears in badges, faucet, and any future multi-round distributor.
