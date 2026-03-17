---
codebase: fatbera-contracts
author: team
chain: Berachain
solidity: "0.8.23"
repo: "0xHoneyJar/fatbera-contracts"
---

# fatBERA Contracts

THJ's Liquid Staking Token (LST). Two-layer ERC4626 vault architecture with multi-token reward streaming.

## Core Architecture

### Two-Layer ERC4626 Vault

```
BERA -> [fatBERA vault] -> fatBERA (base LST)
fatBERA -> [xfatBERA vault] -> xfatBERA (auto-compounding)
```

**Layer 1 -- fatBERA**: Base vault. Deposits BERA, mints fatBERA. 1:1 ratio at genesis, appreciates as rewards accrue. This is the liquid staking receipt.

**Layer 2 -- xfatBERA**: Compounder vault. Deposits fatBERA, mints xfatBERA. Auto-compounds rewards back into the base vault. Users who want passive compounding hold xfatBERA. Users who want to manually manage rewards hold fatBERA.

```solidity
function compound() external onlyRole(OPERATOR_ROLE) {
    // 1. Claim WBERA rewards from fatBERA
    fatBERA.getReward(address(this));
    // 2. Deposit WBERA back into fatBERA
    fatBERA.deposit(wberaBalance, address(this));
    // xfatBERA totalAssets increases -> share price rises
}
```

This separation lets DeFi protocols integrate at either layer depending on their needs.

### Synthetix-Style Multi-Token Reward Streaming

```solidity
uint256 public constant MAX_REWARD_TOKENS = 10;

struct RewardData {
    uint256 rewardsDuration;
    uint256 periodFinish;
    uint256 rewardRate;      // tokens per second
    uint256 lastUpdateTime;
    uint256 rewardPerShareStored;
}

mapping(address rewardToken => RewardData) public rewardData;
```

Up to 10 different reward tokens can be streamed simultaneously to stakers. Each token has independent rate, period, and accounting. Based on the Synthetix StakingRewards pattern but generalized to N tokens.

Rewards are distributed proportionally to vault shares. The `rewardPerShareStored` accumulator pattern avoids iterating over all stakers on every update. Rewards are NOT embedded in the share price -- they are distributed as separate claims.

### Whitelisted Vault Pattern

```solidity
mapping(address => bool) public isWhitelistedVault;
mapping(address user => mapping(address vault => uint256)) public vaultedShares;

function _effectiveBalance(address account) internal view returns (uint256) {
    if (isWhitelistedVault[account]) return 0; // vault doesn't accrue
    return balanceOf(account) + totalVaultedShares[account]; // user keeps accrual
}
```

Only approved DeFi vaults can hold fatBERA without diluting rewards. When a user deposits fatBERA into a whitelisted vault, the vault's balance returns 0 for reward calculation, but the original depositor's effective balance includes their vaulted shares. Rewards follow the user, not the vault contract.

This is a deliberate tradeoff: less permissionless, but protects reward economics for existing depositors. The whitelist is governance-controlled.

### Principal Segregation

```solidity
uint256 public depositPrincipal;

function withdrawPrincipal(uint256 amount) external onlyRole(DEFAULT_ADMIN_ROLE) {
    depositPrincipal -= amount;
    WBERA.withdraw(amount);
    // -> send to validators via AutomatedStake
}
```

Tracks the original deposit amount separately from accumulated rewards. This is critical for validator staking -- the principal must be available for beacon chain deposits, while rewards flow to a different accounting path. `depositPrincipal` is tracked separately from `totalSupply` so admin can move funds to validators while share accounting stays consistent.

### V3 Fee Math Fix

```solidity
// V2 (vulnerable):
uint256 fee = amount * feeRate / 1e18;

// V3 (fixed):
uint256 fee = shares.mulDivUp(feeBps, 10000);
```

`mulDivUp` rounds fees UP instead of down. This prevents the 1-wei exploit where an attacker makes many tiny deposits/withdrawals, each rounding the fee to zero, effectively avoiding fees entirely. Also uses `Math.Rounding.Ceil` for treasury withdrawals.

This is a common vulnerability in vault contracts. Always use `mulDivUp` for fee calculations and `mulDivDown` for share calculations (favor the vault, not the user, on rounding).

### AutomatedStake

Automated process for moving accumulated deposits to the beacon chain deposit contract. Batches deposits to meet the minimum stake requirement.

```solidity
function automatedStake() external onlyKeeper {
    uint256 available = address(this).balance;
    require(available >= MIN_STAKE, "Below minimum");
    // create validator deposit
    depositContract.deposit{value: MIN_STAKE}(pubkey, withdrawalCredentials, signature, depositDataRoot);
}
```

Keeper-triggered (not user-triggered) to batch deposits efficiently and manage validator key rotation.

## Key Takeaways for Future Agents

1. **Two-layer vault** (base + compounder) gives DeFi protocols integration flexibility. The base layer is composable; the compounder layer is set-and-forget.
2. **Synthetix reward streaming** scales to N tokens. Use `rewardPerShareStored` accumulator to avoid O(n) staker iteration.
3. **Whitelist vaults** when reward dilution is a concern. Permissionlessness is not always the right default. Route rewards to users, not vault contracts.
4. **Segregate principal from rewards** in staking systems. They have different destinations and different accounting.
5. **`mulDivUp` for fees, `mulDivDown` for shares**. The 1-wei exploit is real. Always round in the protocol's favor.
6. **Keeper-triggered staking** batches deposits and manages validator lifecycle. Don't let individual users trigger validator operations.
