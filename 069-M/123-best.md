rvierdiiev

medium

# Vault.claim should be called before `pushTotalUnderlyingToController`

## Summary
Vault.claim should be called before `pushTotalUnderlyingToController`.
## Vulnerability Detail
Every cycle, each vault should send it's total underlying amount to xController, so xController can calculate according to new allocations, which amount of underlying vault should receive from/send to xController. Also it [calculates `exchangeRate`](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L307) that will be used fro new depositors/withdrawers.

`MainVault.pushTotalUnderlyingToController` is responsible for sending underlying amount.
It first [calls `setTotalUnderlying`](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L252) function, which will calculate amount of underlying inside all active providers. And then it will calculate total amount by adding funds controlled by vault without reserved funds amount.
`uint256 underlying = savedTotalUnderlying + getVaultBalance() - reservedFunds;` Then this amount is sent to xController. 

Now let's check another function `Vault.claimTokens` which is called inside [`rebalance` function](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L141). The purpose of this function is to receive earned rewards tokens from providers and then [swap them to `vaultCurrency` token](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L408-L417).

Of course, this increases `getVaultBalance()`.

So the main point of this bug is that `Vault.claim` should be called inside `pushTotalUnderlyingToController` as this is also funds that should be then distributed among vaults according to allocations and it should also increased `exhangeRate`.
## Impact
Underlying amount sent to xController is not accurate, actually vault can have more funds. Exhange rate is also not accurate.
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Call `Vault.claim` inside `pushTotalUnderlyingToController`.