evan

high

# Players can call rebalanceBasket before rewards have been pushed to the game

## Summary
It's possible to manipulate game.sol to a state where players can call rebalanceBasket before rewards have been settled. In some cases, it's possible for a malicious user to profit. In other cases, it's possible for a malicious user to create a period where players lose a significant amount of rewards when they call rebalanceBasket.

## Vulnerability Detail
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L343
Observe that vault's receiveProtocolAllocations can be called by the xProvider regardless what state the vault is in.

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L407
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L472
This means that game.sol's pushAllocationsToVaults can be called immediately after pushAllocationsToController.

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L476
PushAllocationsToVaults resets isXChainRebalancing.

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L324
So if someone call PushAllocationsToVaults for vaults on all the chains, players can rebalanceBasket again.

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L442
Vault's rebalancingPeriod on game.sol is incremented in pushAllocationsToController, which is at the very beginning of the rebalancing process.

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L362
The reward for this rebalancingPeriod is not pushed to the game until the very end of the rebalancing process.

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L390
The problem with calling rebalanceBasket before the reward for the current rebalancingPeriod is pushed, is that the value of the currentReward is 0 ([hasn't been set yet](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L526)) so the reward calculation is wrong. Depending on the actual value of currentReward, the user can make a profit or incur a loss.

## Impact
Consider the following scenario. A vault is ready for rebalancing, so the malicious user calls pushAllocationsToController, and then calls pushAllocationsToVaults for vaults on all chains immediately after.

The period from now to when the rebalancing process finishes is a time frame where reward calculation is wrong (currentReward is 0) but rebalanceBasket can still be called. A unaware user can call rebalanceBasket and get a completely different reward than they are supposed to. More often than not, this is unfavorable since currentReward should usually be positive. The length of this period depends on how fast the rebalancing process, which can be delayed by a variety of factors, completes. As discussed in my other reports, there are various ways to interrupt the rebalancing process.

But regardless of how short this period is, the malicious user can predetermine the actual value of currentReward. If it's negative, then they immediately call rebalanceBasket and get a higher reward than they are supposed to.

## Code Snippet
See Vulnerability Detail.

## Tool used

Manual Review

## Recommendation
Prevent rebalanceBasket from being called before rewards for the current period have been settled.
