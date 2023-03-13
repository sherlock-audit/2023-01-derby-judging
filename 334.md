gogo

high

# Wrong type casting leads to unsigned integer underflow exception when current price is < last price

## Summary

When the current price of a locked token is lower than the last price, the Vault.storePriceAndRewards will revert because of the wrong integer casting.

## Vulnerability Detail

The following line appears in Vault.storePriceAndRewards:

```solidity
int256 priceDiff = int256(currentPrice - lastPrices[_protocolId]);
```
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L233

If lastPrices[_protocolId] is higher than the currentPrice, the solidity compiler will revert due the underflow of subtracting unsigned integers because it will first try to calculate the result of `currentPrice - lastPrices[_protocolId]` and __then__ try to cast it to int256.

## Impact

The rebalance will fail when the current token price is less than the last one stored.

## Code Snippet

```solidity
int256 priceDiff = int256(currentPrice - lastPrices[_protocolId]);
```
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L233

## Tool used

Manual Review

## Recommendation

Casting should be performed in the following way to avoid underflow and to allow the priceDiff being negative:

```solidity
int256 priceDiff = int256(currentPrice) - int256(lastPrices[_protocolId]));
```