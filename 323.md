Jeiwan

medium

# Missing transaction expiration check result in reward tokens selling at a lower price

## Summary
Selling of reward tokens misses the transaction expiration check, which may lead to reward tokens being sold at a price that's lower than the market price at the moment of a swap. Gamers and depositors may receive less yield than expected.
## Vulnerability Detail
The protocol integrates with third-party vaults (e.g. Aave, Compound, Yearn, etc.), in which it deposits user funds to generate yield. Some of the third-party vaults (at the moment, only Compound) reward their users with protocol tokens (COMP, in the case of Compound). These tokens can be claimed and sold for the underlying token of the vaults (e.g. USDC):
1. [during rebalancing](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L141);
1. at any moment by calling [claimTokens](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L404).

The [Swap.swapTokensMulti](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/libraries/Swap.sol#L64) function, which is responsible for selling tokens on Uniswap, sets the deadline argument of the `exactInput` call to `block.timestamp`–this basically disables the transaction expiration check because the deadline will be set to whatever timestamp the block including the transaction is minted at.

Transaction expiration check (implemented in Uniswap via the deadline argument) allows users of Uniswap to protect from selling tokens at an outdated price that's lower than the current price. Consider this scenario:
1. The [Vault.rebalance](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L135) function is called on the Ethereum mainnet.
1. Before the transaction is mined, there's a rapid increase of gas cost. The transaction remains in the mempool for some time since the gas cost paid by the transaction is lower than the current gas price.
1. While the transaction is in the mempool, the price of the reward token increases.
1. After a while, gas cost drops and the transaction is mined. Since the value of `amountOutMinimum` was calculated based on an outdated reward token price which is now lower than the current price, the swapping is sandwiched by a MEV bot. The bot decreases the price of the reward token in a Uniswap pool so than the minimum output amount check still holds and earns a profit from the swapping happing at a lower price.
1. As a result of the sandwich attack, reward tokens are swapped at an outdated price, which is now lower than the current price of the tokens. The protocol (and thus gamers and depositors) earn less yield than they could've received by selling the tokens at the current price.
## Impact
Claiming and selling reward tokens can be exploited by a sandwich attack. Gamers and depositors may receive less yield than expected due to reward tokens have been sold at an outdated price.
## Code Snippet
1. `ClaimTokens` sells the reward token for the underlying token:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L404
1. `ClaimToken` is called during rebalancing:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L141
1. The proceedings from selling reward tokens increase the total number of underlying tokens during rebalancing:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L158-L164
1. When `Swap.swapTokensMulti` calls Uniswap's `exactInput`, it sets deadline to `block.timestamp`, which disables the transaction expiration protection:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/libraries/Swap.sol#L88
## Tool used
Manual Review
## Recommendation
Consider a reasonable value to the deadline argument. For example, [Uniswap sets it to 30 minutes on the Etehreum mainnet and to 5 minutes on L2 networks](https://github.com/Uniswap/interface/blob/main/src/constants/misc.ts#L7-L8). Also consider letting the guardian and/or the DAO change the value when on-chain conditions change and may require a different value.