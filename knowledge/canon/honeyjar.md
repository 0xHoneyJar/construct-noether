---
codebase: honeyjar-contracts
author: team
chain: Multi-chain (Ethereum + L2s via LayerZero)
solidity: "0.8.23"
evm: paris
repo: "0xHoneyJar/honeyjar-contracts"
path: "src/"
---

# HoneyJar — Canon Entry

## Architecture

Hub-and-spoke with GameRegistry as the center:

```
GameRegistry (AccessControl)
  └─ all contracts inherit GameRegistryConsumer
  └─ roles: GAME_ADMIN, GAME_INSTANCE, MINTER, BURNER, BEEKEEPER, JANI, GATEKEEPER, PORTAL

HoneyJar (ERC-721)           ← minted by Beekeeper (MINTER), burned by BURNER
Beekeeper                    ← mint orchestrator, pricing, checkpoints, fermentation
Gatekeeper                   ← multi-gate Merkle allow-list
BearPouch                    ← revenue distribution (WAD shares)
Den                          ← prize escrow, releases on fermented jar
```

## Key Patterns

### GameRegistry + GameRegistryConsumer

```solidity
// Registry (hub)
contract GameRegistry is AccessControl {
    mapping(address => bool) public games;

    function startGame(address game_) external onlyRole(Constants.GAME_ADMIN) {
        _grantRole(Constants.GAME_INSTANCE, game_);
        _grantRole(Constants.MINTER, game_);
        games[game_] = true;
    }
}

// Consumer (spoke)
abstract contract GameRegistryConsumer {
    GameRegistry public immutable gameRegistry;

    modifier onlyRole(bytes32 role_) {
        if (!gameRegistry.hasRole(role_, msg.sender)) revert ...;
        _;
    }
}
```

Every contract in the ecosystem inherits `GameRegistryConsumer`. All authorization flows through one hub. Swap a game contract without touching the NFT.

### LibBitmap for Per-Token Boolean State

```solidity
using LibBitmap for LibBitmap.Bitmap;
LibBitmap.Bitmap private fermented;

function isFermented(uint256 tokenId) external view returns (bool) {
    return fermented.get(tokenId);
}

function ferment(uint256 tokenId) external onlyRole(Constants.GAME_INSTANCE) {
    fermented.set(tokenId);
}
```

1 bit per tokenId. 10,000 tokens = ~40 storage slots. Vs. `mapping(uint256 => bool)` = 10,000 storage slots.

### Role-Gated Burn (Unassigned Until Activated)

```solidity
// In HoneyJar.sol:
function burn(uint256 _id) external override onlyRole(Constants.BURNER) {
    _burn(_id);
}
```

BURNER role is intentionally empty at deploy. Nothing can burn until the admin explicitly grants BURNER to a crafting/game contract. This is a safety pattern: the capability exists in the type system but is not realized until the system is ready.

### Staged URI Reveal

```solidity
bool public isGenerated;

function tokenURI(uint256 tokenId) public view override returns (string memory) {
    if (!_exists(tokenId)) revert URIQueryForNonexistentToken();
    string memory baseURI = _baseURI();
    return isGenerated ? string.concat(baseURI, tokenId.toString()) : baseURI;
}
```

Pre-reveal: all tokens return the same base URI (mystery/placeholder). Post-reveal: each token gets its own URI (`baseURI + tokenId`). Toggle controlled by `realOwner` (multisig).

### MultisigOwnable (Dual Ownership)

```solidity
abstract contract MultisigOwnable is Ownable {
    address public realOwner;

    modifier onlyRealOwner() {
        require(realOwner == msg.sender);
        _;
    }

    function transferLowerOwnership(address newOwner) public onlyRealOwner {
        _transferOwnership(newOwner);
    }
}
```

Two owners: `realOwner` (multisig, immutable admin) and OZ `owner` (EOA, for OpenSea collection settings). `realOwner` can reclaim the lower owner at any time. Solves the OpenSea-requires-EOA problem without sacrificing multisig security.

### Checkpoint-Based Fermentation

```solidity
// In Beekeeper._mintHoneyJar():
uint256 numMinted = progress.mintedJars.length;
if (numMinted >= progress.checkpoints[progress.checkpointIndex]) {
    _fermentJars();  // triggers randomness request
}
```

Checkpoints are mint thresholds (e.g., [50, 100, 150]). When total mints cross a checkpoint, a winning jar is randomly selected and "fermented" (flagged via LibBitmap). This creates anticipation without explicit lottery UX.

### WAD-Based Fee Distribution

```solidity
// In BearPouch:
function _updateDistributions(DistributionConfig[] memory _distributions) internal {
    uint256 shareSum = 0;
    for (uint256 i = 0; i < _distributions.length; i++) {
        shareSum += _distributions[i].share;
        distributions.push(_distributions[i]);
    }
    if (shareSum != FixedPointMathLib.WAD) revert InvalidDistributionConfig(shareSum);
}

// In distribute():
paymentToken.safeTransferFrom(msg.sender, distributions[i].recipient, amountERC20.mulWad(distributions[i].share));
```

Shares must sum to 1e18 (WAD). `mulWad` handles the math. Cleaner than basis points for multi-recipient splits.

## Dependencies

- Solady: `SafeTransferLib`, `FixedPointMathLib`, `ReentrancyGuard`, `MerkleProofLib`, `SafeCastLib`, `LibBitmap`
- OZ: `AccessControl`, `ERC721`, `IERC20`, `SafeERC20`
- LayerZero: `solidity-examples` (ONFT721 for cross-chain)
- `foundry.toml`: `evm_version = "paris"`, `auto_detect_solc = true`

## Operational Note

The HJ contracts were structurally sound but difficult to operate in production. The registry pattern is excellent; the operational complexity came from cross-chain coordination and checkpoint management. In a simpler (single-chain, no fermentation) context, the patterns are clean and effective.
