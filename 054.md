wzrdk3lly

high

# Unprotected slippage tolerance can lead to user/protocol loss of funds

## Summary

Slippage is the difference between the expected price of an order and the price when the order actually executes. If the slippage tolerance is set too low, a transaction will not execute. If the slippage tolerance is set too high, users will lose money for paying more per token than intended during swaps. When a rebalance is ready to take place on Derby, any user can front run the rebalancXChain() call setting the slippage tolerance to be 100% for the xChain swap.

## Vulnerability Detail

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L307-L320

1. Any external user/attacker invokes the `rebalanceXChain()` by frontrunning the transaction when it hits the mempool

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L321-L333

2. When the `homechain` is not equal to the `controllerChain` `xTransfer()` is called with the `_slippage` tolerance

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L152-L159

3. This XCall will cause the rebalance to be performed, making the proper XChain swaps with the unprotected slippage tolerance.

## Impact

Fulfilled swaps where the actual slippage is significantly higher than the standard tolerance of 30 (.03%) would result in loss of funds. 

## Code Snippet

The entrypoint for this attack occurs in the `rebalanceXChain()` function below
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L307-L320

## Tool used

Manual Review

## Recommendation

Require minimum and maximum values that the protocol is willing to accept for a slippage tolerance that will not cause significant fund loss during Connext XChain swaps. Alternatively the rebalancXChain() call can leverage onlyDao or onlyKeeper modifiers to ensure that these are the only two trusted entities allowed to make this call as intended per the [Derby Documentation](https://derby-finance.gitbook.io/derby-finance-docs/products/vaults/rebalancing). 
