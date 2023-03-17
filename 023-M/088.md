zaevlad

high

# Malicious user can manipulate exchange rate to get more profit

## Summary

Malicious user can manipulate exchange rate to exaggerated profit in several times due to a lack of checks.

## Vulnerability Detail

Initial exchange rate is going to be equal to the decimals of Vault currency. Here is a quote from the docs: Where the exchangeRate is expressed in underlying/ LPtoken in the scale of the underlying vault currency and LPScale is the amount of decimals of the LP token in the same scale (10^6).

So let’s take USDT as an example. 

1. 3 regular users deposit 100 USDT to the Vault and get 100 shares (lp tokens) based on the calculations: shares = (amount * (10 ** decimals())) / exchangeRate;
2. A malicious user deposit 10 000 USDT and get 10 000 lp tokens.
3. A Vault has 10 300 USDT as totalSupply and 10 300 lp tokens as minted.
4. Right after out malicious user call withdrawalRequest() and burn his lp tokens. 
5. Also he deposit 200 USDT again.
6. After all a Vault has 10 500 USDT and 500 lp tokens. 
7. Later the rebalancing period will modify exchangeRate based on the calculations: uint256 newExchangeRate = (totalUnderlying * (10 ** decimals)) / totalSupply;
8. Our new exchangeRate will be (500*(10**6))/10500 = 47619
9. And users can withdraw much more tokens than expected. For example, a regular user can take (100*47619) / (10*6) = 79365 of USDT from the Vault.   


## Impact

During the next rebalancing period a malicious user can take as much collateral tokens as he want and leaves a Vault with no assets.

## Code Snippet

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L121
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L149
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L307

## Tool used

Manual Review

## Recommendation

Add ckecks to withdrawalRequest() if a Vault has enough assets right now a user will be redirected to withdraw() immediately .
