---
codebase: fatbera-contracts
author: team
chain: Berachain
solidity: "0.8.23"
repo: "0xHoneyJar/fatbera-contracts"
---

# fatBERA — Canon Entry

## What It Is

THJ's Liquid Staking Token. Two-layer ERC-4626 vault system: fatBERA (base LST with multi-token rewards) + xfatBERA (auto-compounding wrapper). Plus automated validator staking.

## Key Patterns

### Synthetix-Style Multi-Token Reward Streaming

```solidity
struct RewardData {
    uint256 rewardsDuration;
    uint256 periodFinish;
    uint256 rewardRate;
    uint256 lastUpdateTime;
    uint256 rewardPerShareStored;
}
mapping(address rewardToken => RewardData) public rewardData;
```

Up to 10 reward tokens simultaneously. Rewards stream over `rewardsDuration`. `rewardPerShareStored` accumulates. NOT embedded in share price — distributed as separate claims.

### Whitelisted Vault Pattern (DeFi Composability)

```solidity
mapping(address => bool) public isWhitelistedVault;
mapping(address user => mapping(address vault => uint256)) public vaultedShares;

function _effectiveBalance(address account) internal view returns (uint256) {
    if (isWhitelistedVault[account]) return 0; // vault doesn't accrue
    return balanceOf(account) + totalVaultedShares[account]; // user keeps accrual
}
```

Users can deposit fatBERA into DeFi (whitelisted vaults) without losing reward accrual. The rewards follow the original depositor, not the vault.

### Layered Compounding (xfatBERA)

```solidity
function compound() external onlyRole(OPERATOR_ROLE) {
    // 1. Claim WBERA rewards from fatBERA
    fatBERA.getReward(address(this));
    // 2. Deposit WBERA back into fatBERA
    fatBERA.deposit(wberaBalance, address(this));
    // xfatBERA totalAssets increases → share price rises
}
```

Layer 2 vault auto-compounds Layer 1 rewards. Each xfatBERA share is worth more fatBERA over time.

### V3 Fee Math Fix

```solidity
// V2 bug: double-fee on withdrawal
// V3 fix: use mulDivUp to round fee up (favoring vault)
uint256 fee = shares.mulDivUp(feeBps, 10000);
// Also: Math.Rounding.Ceil for treasury withdrawals to prevent 1-wei exploit
```

### Principal Segregation

```solidity
uint256 public depositPrincipal;

function withdrawPrincipal(uint256 amount) external onlyRole(DEFAULT_ADMIN_ROLE) {
    depositPrincipal -= amount;
    WBERA.withdraw(amount);
    // → send to validators via AutomatedStake
}
```

`depositPrincipal` tracked separately from `totalSupply`. Admin can move funds to validators while share accounting stays consistent.
