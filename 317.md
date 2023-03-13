Jeiwan

high

# Delayed Compound and Beta interest accrual reduces gamer rewards and affects funds distribution to vaults

## Summary
During rebalancing, Compound and Beta interest is not accrued until after gamer rewards are calculated leading to the rewards being calculated on the underlying amounts that are smaller than the real amounts. As a result, gamers receive reduced rewards. Also, since interest accrual doesn't happen when vaults push underlying amounts to `XChainController`, the underlying amounts to be distributed among vaults will be smaller that real amounts.
## Vulnerability Detail
The Derby protocol allows users to deposit funds to third-party protocols to earn passive yield. Two of such protocols are Compound and Beta: funds deposited to Compound and Beta earn interest collected from borrowers who borrow the funds. Due to accruing of interest, the exchange rate of cTokens and BTokens increases over time: for example, 1 cToken can be exchange for 1.01 underlying token after some time.

During rebalancing, Derby vaults report their underlying token balances (the amount of funds deposited in third-party protocols) to `XChainController`, which re-allocates and re-distributes them according to the allocations set by gamers. However, when the underlying balances are calculated, interest is not accrued on Compound and Beta:
1. [setTotalUnderlying](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L252) calls [balanceUnderlying](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L344) on each of the supported protocol;
1. [CompoundProvider.balanceUnderlying](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L90) multiplies contract's cToken balance by the exchange rate of the cToken;
1. [CompoundProvider.exchangeRate](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L127) calls [ICToken.exchangeRateStored](https://github.com/compound-finance/compound-protocol/blob/master/contracts/CToken.sol#L279-L286), which doesn't accrue interest and returns cached exchange rate:
> This function does not accrue interest before calculating the exchange rate
1. Likewise, [BetaProvider.balanceUnderlying](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/BetaProvider.sol#L82) and [BetaProvider.calcShares](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/BetaProvider.sol#L99) read `IBeta.totalLoanable` and `IBeta.totalLoan` to convert BTokens to underlying tokens, however the interest is not accrued in `totalLoan` beforehand (as can be seen in the code of `BToken`, [accruing interest increases `totalLoan`](https://github.com/beta-finance/beta/blob/master/contracts/BToken.sol#L92-L93)).

Thus, the interest accrued by the funds deposited to Compound and Beta since the previous rebalancing won't be counted in the new rebalancing, and the underlying balance reported by the vault will be lower than the real balance.

Interest is accrued only at the end of rebalancing, in the [Vault.rebalance](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L135) function: first [deposit](https://github.com/compound-finance/compound-protocol/blob/master/contracts/CToken.sol#L387) or [withdrawal](https://github.com/compound-finance/compound-protocol/blob/master/contracts/CToken.sol#L457) to/from Compound and Beta will accrue interest. However, this will happen after gamer rewards have been calculated:
1. gamer rewards are calculated in [storePriceAndRewards](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L186), before funds are deposited/withdrawn to Compound or Beta;
1. gamer rewards are calculated based on the underlying balance calculated in [calcUnderlyingIncBalance](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L158), which reads the value of `savedTotalUnderlying`–it was set in the beginning of rebalancing, and interest wasn't accrued before it was set.

Thus, gamer rewards will always lag behind actual underlying balances of Compound and Beta, and gamers will always earn reduced rewards.
## Impact
Gamer receive reduced rewards due to delayed accruing of interest in Compound and Beta.
## Code Snippet
1. `setTotalUnderlying` is called when vaults report their balances:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L252
1. `setTotalUnderlying` calls `balanceUnderlying`, which calls `balanceUnderlying` on each protocol provider:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L344-L361
1. `CompoundProvider.balanceUnderlying` calls `exchangeRate` to get the exchange rate of the cToken:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L96
1. The exchange rate is read via the `exchangeRateStored` function, which returned cached rate: the interest earned since the previous accrual is not counted:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L127
https://github.com/compound-finance/compound-protocol/blob/master/contracts/CToken.sol#L284-L293
1. `BetaProvider.balanceUnderlying` and `BetaProvider.calcShares` read `IBeta.totalLoan` to convert BTokens to underlying tokens:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/BetaProvider.sol#L89
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/BetaProvider.sol#L102
1. In `BToken`, `totalLoan` is increased by accumulated interest when interest is accrued:
https://github.com/beta-finance/beta/blob/master/contracts/BToken.sol#L92-L93
## Tool used
Manual Review
## Recommendation
In `CompoundProvider.exchangeRate`, consider calling [exchangeRateCurrent](https://github.com/compound-finance/compound-protocol/blob/master/contracts/CToken.sol#L274) instead of `exchangeRateStored`.
In `BetaProvider.balanceUnderlying` and `BetaProvider.calcShares` consider calling `IBeta.accrue` before calling `IBeta.totalLoan`.