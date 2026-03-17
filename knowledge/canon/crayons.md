---
codebase: crayons-monorepo
author: exp
chain: Berachain
solidity: "0.8.26"
evm: cancun
repo: "0xHoneyJar/crayons-monorepo"
path: "contracts/crayons-contracts/src/"
---

# Crayons — Canon Entry

## Architecture

Three-layer Beacon system:

```
Factory (non-upgradeable, Solady Ownable)
  └─ deploys BeaconProxy instances via CREATE2
       └─ all point to ERC721Base_beacon (upgradeable implementation)

ControllerV1 (UUPS upgradeable, OZ Ownable)
  └─ holds fee split config
  └─ distribute(token, amount, creator) called atomically during mint
```

## Key Patterns

### ERC-7201 Namespaced Storage

```solidity
/// @custom:storage-location erc7201:honeyjar.storage.ERC721Base
struct ERC721BaseStorage {
    string baseURI;
    address currency;
    address controller;
    uint128 totalSupply;
    uint128 price;
    uint128 maxSupply;     // 0 => open edition
    bool    isUnique;      // true => unique token (id 0)
}

bytes32 private constant ERC721BaseStorageLocation =
    0xed0138aeb49f7f80b0702b7b725ce7e0e4e27f3c04a41055673c936c60eb3700;

function _getERC721BaseStorage() private pure returns (ERC721BaseStorage storage $) {
    assembly { $.slot := ERC721BaseStorageLocation }
}
```

Prevents storage collisions across proxy upgrades. The slot hash is derived from `keccak256("honeyjar.storage.ERC721Base") - 1`.

### Three Edition Modes From Two Flags

```
maxSupply=0, isUnique=false  → Open Edition (unlimited)
maxSupply=N, isUnique=false  → Limited Edition (capped at N)
maxSupply=1, isUnique=true   → Unique (1-of-1, always tokenId 0)
```

One contract, three behaviors. No inheritance hierarchy needed.

### Launch Lifecycle

```solidity
function initialize(...) external initializer {
    // ... setup ...
    _pause();  // starts paused
}

function launch(uint128 _price) public virtual onlyOwner {
    ERC721BaseStorage storage $ = _getERC721BaseStorage();
    $.price = _price;
    _unpause();
    emit Launched();
}
```

Deploy → configure (setBaseURI, setRoyalty, etc.) → launch. No race conditions.

### Fee Distribution Atomic With Mint

```solidity
function _mint(address to, uint256 tokenId, uint128 price) internal {
    ERC721BaseStorage storage $ = _getERC721BaseStorage();
    if (price > 0) {
        STL.safeTransferFrom($.currency, msg.sender, $.controller, price);
        ControllerV1($.controller).distribute($.currency, price, owner());
    }
    super._mint(to, tokenId);
    $.totalSupply++;
}
```

Payment and distribution happen BEFORE the token is minted. Three-way split: platform, creator, POL (protocol-owned liquidity). Basis points (10000 denominator).

### Beacon Proxy Factory

```solidity
function createERC721Base(...) external returns (address) {
    bytes memory data = abi.encodeWithSelector(ERC721Base.initialize.selector, ...);
    address erc721Base = address(new BeaconProxy(ERC721Base_beacon, data));
    emit Factory__NewERC721Base(_owner, erc721Base);
    return erc721Base;
}
```

One-click deploy. All proxies share the same implementation (upgradeable via beacon). Near-zero marginal cost per collection.

## Dependencies

- Solady: `SafeTransferLib`, `FixedPointMathLib`, `Ownable`
- OZ Upgradeable: `ERC721Upgradeable`, `OwnableUpgradeable`, `PausableUpgradeable`, `UUPSUpgradeable`
- `foundry.toml`: `evm_version = "cancun"`, `optimizer_runs = 200`, `extra_output = ["storageLayout"]`
