---
codebase: interpol
author: team
chain: Berachain
solidity: "0.8.23"
repo: "0xHoneyJar/interpol"
---

# Interpol — Canon Entry

## What It Is

LP lockers on Berachain ("rug insurance"). Per-user locker instances deployed via BeaconProxy factory. Adapters for protocol-agnostic DeFi staking. Badge-based fee discounts.

## Key Patterns

### Double Beacon Proxy

```
LockerFactory → deploys HoneyLocker instances via BeaconProxy
    HoneyLocker → deploys adapters via BeaconProxy from protocol-specific beacons
```

Upgrade ALL lockers by changing one beacon. Upgrade ALL adapters for a protocol by changing its beacon. Two-level upgrade cascade.

### Adapter/Strategy Pattern

```solidity
abstract contract BaseVaultAdapter {
    function stake(address vault, uint256 amount) external virtual;
    function unstake(address vault, uint256 amount) external virtual;
    function claim(address vault) external virtual;
    function wildcard(address vault, uint8 func, bytes calldata args) external virtual;
}
```

4-method interface. Locker is protocol-agnostic. New DeFi protocols added by deploying a new adapter beacon.

### Wildcard Escape Hatch

`wildcard(vault, uint8 func, bytes args)` — generic function dispatch for adapter-specific operations. Allows adapters to expose protocol-unique functionality without changing the base interface.

### Badge-Based Fee Discounts

```solidity
function computeFees(address user, uint256 amount) public view returns (uint256) {
    uint256 discount = badges.badgesPercentageOfUser(user) * 69 / 10000;
    return amount * (baseFee - discount) / 10000;
}
```

NFT badge holdings → linear fee reduction (up to 69%). Bridges NFT collection status to DeFi economics.
