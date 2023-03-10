rvierdiiev

high

# Game doesn't accrued rewards for previous rebalance period in case if rebalanceBasket is called in next period

## Summary
Game doesn't accrued rewards for previous rebalance period in case if `rebalanceBasket` is called in next period. Because of that user do not receive rewards for the previous period and in case if he calls `rebalanceBasket` each rebalance period, he will receive rewards only for last one.
## Vulnerability Detail
When `Game.rebalanceBasket` is called, then basket rewards are accrued [by calling `addToTotalRewards` function](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L327).
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L368-L401
```solidity
  function addToTotalRewards(uint256 _basketId) internal onlyBasketOwner(_basketId) {
    if (baskets[_basketId].nrOfAllocatedTokens == 0) return;


    uint256 vaultNum = baskets[_basketId].vaultNumber;
    uint256 currentRebalancingPeriod = vaults[vaultNum].rebalancingPeriod;
    uint256 lastRebalancingPeriod = baskets[_basketId].lastRebalancingPeriod;


    if (currentRebalancingPeriod <= lastRebalancingPeriod) return;


    for (uint k = 0; k < chainIds.length; k++) {
      uint32 chain = chainIds[k];
      uint256 latestProtocol = latestProtocolId[chain];
      for (uint i = 0; i < latestProtocol; i++) {
        int256 allocation = basketAllocationInProtocol(_basketId, chain, i) / 1E18;
        if (allocation == 0) continue;


        int256 lastRebalanceReward = getRewardsPerLockedToken(
          vaultNum,
          chain,
          lastRebalancingPeriod,
          i
        );
        int256 currentReward = getRewardsPerLockedToken(
          vaultNum,
          chain,
          currentRebalancingPeriod,
          i
        );
        baskets[_basketId].totalUnRedeemedRewards +=
          (currentReward - lastRebalanceReward) *
          allocation;
      }
    }
  }
```
This function allows user to accrue rewards only when `currentRebalancingPeriod > lastRebalancingPeriod`.
When user allocates, he allocates for the [next period](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L230). And `lastRebalancingPeriod` is changed after `addToTotalRewards` is called, so after rewards for previous period accrued. 
And when allocations are sent to the xController, then new rebalance period [is started](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L442). So actually rewards accruing for period that user allocated for is started once `pushAllocationsToController` is called. 
And at this point `currentRebalancingPeriod == lastRebalancingPeriod` which means that if user will call rebalanceBasket for next period, the rewards will not be accrued for him, but `lastRebalancingPeriod` will be incremented. So actually he will not receive rewards for previous period.

Example.
1.`currentRebalancingPeriod` is 10.
2.user calls `rebalanceBasket` with new allocation and `lastRebalancingPeriod` is set to 11 for him.
3.`pushAllocationsToController` is called, so `currentRebalancingPeriod` becomes 11.
4.`settleRewards` is called, so rewards for the 11th cycle are accrued.
5.now user can call `rebalanceBasket` for the next 12th cycle. `addToTotalRewards` is called, but `currentRebalancingPeriod == lastRebalancingPeriod == 11`, so rewards were not accrued for 11th cycle
6.new allocations is saved and `lastRebalancingPeriod` becomes 12.
7.the loop continues and every time when user allocates for next rewards his `lastRebalancingPeriod` is increased, but rewards are not added.
8.user will receive his rewards for previous cycle, only if he skip 1 rebalance period(he doesn't allocate on that period).

As you can see this is very serious bug. Because of that, player that wants to adjust his allocation every rebalance period will loose all his rewards.
## Impact
Player looses all his rewards
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
First of all, you need to allows to call `rebalanceBasket` only once per rebalance period, before new rebalancing period started and allocations are sent to xController. Then you need to change check inside `addToTotalRewards` to this `if (currentRebalancingPeriod < lastRebalancingPeriod) return;` in order to allow accruing for same period.