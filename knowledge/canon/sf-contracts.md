---
codebase: sf-contracts-v2
author: team
chain: Berachain
solidity: "0.8.23"
repo: "0xHoneyJar/sf-contracts-v2"
---

# SF Contracts V2 — Canon Entry

## What It Is

ERC-4626 vault + strategy framework. Pluggable strategies with module composition. Profit unlock anti-sandwich. Badge-based fee rebates converting to HENLO.

## Key Patterns

### Profit Unlock (Anti-Sandwich)

```solidity
function totalAssets() public view returns (uint256) {
    return strategy.totalAssets() - _calculateLockedProfit();
}

function _calculateLockedProfit() internal view returns (uint256) {
    uint256 elapsed = block.timestamp - lastHarvestTimestamp;
    if (elapsed >= profitUnlockTime) return 0;
    return lockedProfit - (lockedProfit * elapsed / profitUnlockTime);
}
```

After harvest, profits lock and release linearly over 6 hours. Prevents sandwich attacks around harvest events. `totalAssets()` subtracts locked profit.

### Module Composition for Strategies

```solidity
contract BeradromeStrategy is
    BaseStrategy,
    BeradromeModule,
    MultiRewardsModule,
    BgtWrapperModule
{
    // Each module provides: init, state, methods
    // Strategy composes capabilities via multiple inheritance
}
```

### Protected Token System

Admin-overridable + strategy-default token protection. Prevents accidental sweeping of receipt/reward tokens during migration while allowing admin flexibility.

### HENLO as Universal Rebate Currency

```solidity
function getRebate(address token, uint256 amount) internal {
    uint256 rebateAmount = amount * badgeDiscount / 10000;
    // Swap rebate portion to HENLO via KXRouter
    router.swap(token, HENLO, rebateAmount);
    // Send to user
}
```

All fee rebates convert to HENLO. Creates constant buy pressure. Badge holdings determine rebate percentage.
