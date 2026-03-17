---
codebase: badges-contracts
author: exp
chain: Berachain
solidity: "0.8.23"
repo: "0xHoneyJar/badges-contracts"
---

# Badges Contracts

Soulbound badge NFTs with multi-round Merkle claiming and NFT-gated tiered pricing.

## Core Architecture

### LibBitmap per Merkle Root

Each Merkle root gets its own claim bitmap. This allows multiple distribution rounds without redeploying or migrating state.

```solidity
mapping(bytes32 => LibBitmap.Bitmap) private _claimedBitmaps;
```

Keyed by Merkle root, not by round number. When a new round starts, a new root is published and claims begin against a fresh bitmap. Old roots remain valid until explicitly deprecated.

### Per-Badge Merkle Root Architecture

Each badge ID maintains its own set of Merkle roots. This means badge #1 can be in round 3 while badge #7 is still in round 1. Independent claim lifecycles per badge.

```solidity
mapping(uint256 badgeId => bytes32[] roots) public badgeRoots;
```

### Soulbound Enforcement

Transfer restriction via `_update` override. Clean pattern -- catches all transfer vectors (transferFrom, safeTransferFrom, burn-and-remint) in one place.

```solidity
function _update(address to, uint256 tokenId, address from) internal override returns (address) {
    if (from != address(0)) revert NotTransferable();
    return super._update(to, tokenId, from);
}
```

`from != address(0)` allows minting but blocks all transfers. Simpler and more gas-efficient than per-function overrides.

## Claiming Patterns

### Selective Mint from Full Entitlement

Users are entitled to multiple badges per proof but can choose which to mint. The `_indexesToMint` pattern lets callers cherry-pick from their full entitlement.

```solidity
function claim(
    bytes32[] calldata proof,
    uint256[] calldata entitledBadgeIds,
    uint256[] calldata _indexesToMint
) external {
    // verify proof against full entitlement
    // only mint the indexes caller selected
}
```

This avoids forcing users to mint everything (saves gas) while keeping the Merkle tree simple (one leaf per user with full entitlements).

### Idempotent Batch Operations

Batch operations skip already-claimed entries instead of reverting. Critical for UX -- users can retry failed transactions without tracking which items succeeded.

```solidity
// skip instead of revert
if (LibBitmap.get(_claimedBitmaps[root], index)) continue;
```

## Pricing & Access

### NFT-Gated Tiered Pricing

Holders of specific NFTs get discounted pricing. Non-holders pay full price. Overpayment is refunded automatically.

```solidity
if (msg.value > requiredPayment) {
    (bool refunded,) = msg.sender.call{value: msg.value - requiredPayment}("");
    require(refunded, "Refund failed");
}
```

Always refund overpayment. Never hold excess ETH. This prevents stuck funds and improves trust.

## Library Strategy

### Hybrid Solady + OpenZeppelin

- **Solady**: LibBitmap, LibString, gas-critical paths. Chosen for raw efficiency.
- **OpenZeppelin**: ERC1155Upgradeable, UUPSUpgradeable, AccessControlUpgradeable. Chosen for battle-tested upgrade infrastructure.

The split is deliberate: Solady where gas matters (claim hot path), OZ where correctness and upgradeability matter (proxy infra).

### UUPS Storage Gap Discipline

```solidity
uint256[50] private __gap;
```

Standard storage gap in every base contract. Ensures future upgrades can add state variables without colliding with derived contract storage.

## Evolution Pattern

```
V1 (testnet) -> V2 (testnet iteration) -> Mainnet (production) -> V2M (mainnet upgrade)
```

Each version is a separate contract with its own deployment. UUPS proxy allows in-place upgrades within a version lineage. The V1->V2->Mainnet->V2M progression shows a typical testnet-to-mainnet hardening path.

## Key Takeaways for Future Agents

1. **LibBitmap keyed by root** is the canonical pattern for multi-round Merkle claims. Do not key by round number.
2. **Soulbound via `_update`** is cleaner than per-function transfer blocks. One override covers all vectors.
3. **Skip-not-revert** in batch operations. Always. Users will retry.
4. **Refund overpayment** in any payable function. Never hold excess.
5. **Solady for gas, OZ for infra** is a valid hybrid approach. Document which library covers what.
