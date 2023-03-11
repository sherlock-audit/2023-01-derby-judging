rvierdiiev

medium

# User should not receive rewards for the rebalance period, when protocol was blacklisted, because of unpredicted behaviour of protocol price

## Summary
User should not receive rewards for the rebalance period, when protocol was blacklisted, because of unpredicted behaviour of protocol price.
## Vulnerability Detail
When user allocates derby tokens to some underlying protocol, he receive rewards according to the exchange price of that protocols token. This reward can be positive or negative.
Rewards of protocol are set to `Game` contract inside [`settleRewards` function](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L498-L504) and they are accumulated for user, once he calls `rebalanceBasket`.

Let's check how they are calculated.
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L226-L245
```solidity
  function storePriceAndRewards(uint256 _totalUnderlying, uint256 _protocolId) internal {
    uint256 currentPrice = price(_protocolId);
    if (lastPrices[_protocolId] == 0) {
      lastPrices[_protocolId] = currentPrice;
      return;
    }


    int256 priceDiff = int256(currentPrice - lastPrices[_protocolId]);
    int256 nominator = (int256(_totalUnderlying * performanceFee) * priceDiff);
    int256 totalAllocatedTokensRounded = totalAllocatedTokens / 1E18;
    int256 denominator = totalAllocatedTokensRounded * int256(lastPrices[_protocolId]) * 100; // * 100 cause perfFee is in percentages


    if (totalAllocatedTokensRounded == 0) {
      rewardPerLockedToken[rebalancingPeriod][_protocolId] = 0;
    } else {
      rewardPerLockedToken[rebalancingPeriod][_protocolId] = nominator / denominator;
    }


    lastPrices[_protocolId] = currentPrice;
  }
```
Every time, previous price of protocol is compared with current price.

In case if some protocol is hacked, there is `Vault.blacklistProtocol` function, that should withdraw reserves from protocol and mark it as blacklisted. 
The problem is that because of the hack it's not possible to determine what will happen with exhange rate of protocol. It can be 0, ot it can be very small or it can be high for any reasons.
But protocol still accrues rewards per token for protocol, even that it is blacklisted. Because of that, user that allocated to that protocol can face with accruing very big negative or positive rewards. Both this cases are bad.

So i believe that in case if protocol is blacklisted, it's better to set rewards as 0 for it.

Example.
1.User allocated 100 derby tokens for protocol A
2.Before `Vault.rebalance` call, protocol A was hacked which made it exchangeRate to be not real.
3.Derby team has blacklisted that protocol A.
4.`Vault.rebalance` is called which used new(incorrect) exchangeRate of protocol A in order to calculate `rewardPerLockedToken`
5.When user calls rebalance basket next time, his rewards are accumulated with extremely high/low value.
## Impact
User's rewards calculation is unpredictable.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
In case if protocol is blacklisted, then set `rewardPerLockedToken` to 0 inside `storePriceAndRewards` function.