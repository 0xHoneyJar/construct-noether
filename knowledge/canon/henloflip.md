---
codebase: henloflip-contracts
author: team
chain: Berachain bArtio
solidity: "0.8.19"
repo: "0xHoneyJar/henloflip-contracts"
---

# HenloFlip

Coin-flip gambling game with progressive jackpot. Uses Pyth Entropy for VRF instead of Chainlink.

## Core Architecture

### Pyth Entropy VRF

Pyth Entropy is a callback-based VRF. The contract requests randomness, pays a fee, and receives the result in a callback.

```solidity
function flip(uint256 tier) external payable {
    // collect wager
    uint64 sequenceNumber = entropy.requestWithCallback{value: entropy.getFee(provider)}(
        provider,
        userRandomNumber
    );
    // store pending game keyed by sequenceNumber
}

function entropyCallback(uint64 sequenceNumber, address, bytes32 randomNumber) internal override {
    // resolve game using randomNumber
}
```

Key difference from Chainlink VRF: Pyth uses a provider model where you choose your entropy provider. The fee is queried dynamically via `getFee()`.

### Progressive Jackpot

The jackpot probability increases with consecutive losses. Not per-user -- global loss counter.

```solidity
uint256 public lossCount;

function _checkJackpot(uint256 randomNumber) internal returns (bool) {
    // jackpot probability = lossCount / 1000
    uint256 jackpotRoll = randomNumber % 1000;
    if (jackpotRoll < lossCount) {
        // JACKPOT -- reset counter, pay out
        lossCount = 0;
        return true;
    }
    lossCount++;
    return false;
}
```

After 1000 consecutive losses without jackpot, probability reaches 100%. The jackpot is guaranteed to hit eventually. This creates a known upper bound on jackpot accumulation.

### Fixed Wager Tiers

```solidity
uint256[3] public WAGER_TIERS = [33 ether, 69 ether, 420 ether]; // HENLO tokens
```

No arbitrary amounts. Three fixed tiers. This simplifies pot accounting and prevents dust attacks. The numbers (33, 69, 420) are meme-native -- they match the community's culture.

### Dual-Modulus Single Random Number

One 256-bit random number serves both the coin flip and jackpot determination.

```solidity
function _resolveGame(bytes32 randomNumber) internal {
    uint256 rng = uint256(randomNumber);
    bool won = (rng % 2) == 0;         // coin flip: mod 2
    bool jackpot = _checkJackpot(rng);   // jackpot: mod 1000
}
```

The same random number feeds two independent outcomes via different moduli. `mod 2` and `mod 1000` are coprime, so the outcomes are statistically independent. One VRF call, two game mechanics. Saves gas and VRF fees.

### Game Pot Cap with Treasury Overflow

The game pot has a maximum size. When wagers push it over the cap, excess flows to the treasury.

```solidity
if (gamePot + wager > GAME_POT_CAP) {
    uint256 overflow = (gamePot + wager) - GAME_POT_CAP;
    treasury += overflow;
    gamePot = GAME_POT_CAP;
} else {
    gamePot += wager;
}
```

This prevents the game pot from growing unboundedly (which would make the jackpot too attractive and destabilize the economy). Treasury overflow funds protocol operations.

### Event-Only Factory

QuestsFactory deploys quest contracts and emits events. The factory itself holds no state about deployed quests. The blockchain event log IS the registry.

```solidity
contract QuestsFactory {
    event QuestDeployed(address quest, bytes32 questId);

    function deploy(bytes32 questId, bytes calldata initData) external returns (address) {
        address quest = _deploy(initData);
        emit QuestDeployed(quest, questId);
        return quest;
    }
}
```

Indexers reconstruct the quest registry from events. On-chain, the factory is stateless after deployment. This is the "blockchain as event bus" pattern -- cheaper than maintaining on-chain mappings for data that's only read off-chain.

## Key Takeaways for Future Agents

1. **Pyth Entropy** is a viable VRF alternative to Chainlink. Callback pattern, provider model, dynamic fees. Consider it for Berachain deployments.
2. **Progressive jackpot via loss counter** creates guaranteed-hit mechanics. `lossCount/1000` means jackpot hits within 1000 rounds maximum.
3. **Fixed wager tiers** simplify accounting and prevent edge cases. Use culturally resonant numbers.
4. **Dual-modulus on single VRF** saves gas. As long as moduli are coprime, outcomes are independent.
5. **Event-only factories** are cheaper than stateful registries when consumers are off-chain indexers.
6. **Pot caps with treasury overflow** prevent unbounded accumulation in gambling contracts.
