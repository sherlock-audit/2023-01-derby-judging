rvierdiiev

medium

# exchangeRate is not up to date in case if vault that is off is changed to on

## Summary
`exchangeRate` is not up to date in case if vault that is off is changed to on. Depositors and withdrawers will use old `exchangeRate` that will be less than in reality.
## Vulnerability Detail
In case if vault has no allocations, then it is [switched off](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L230). That means that users can't deposit/withdraw from vault anymore. After it was detected that vault is off or on, then `sendFeedbackToVault` [is called](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L247-L255).

In case if vault is off, that means that actually it doesn't have underlying inside providers anymore, so there is [no reason send `exchangeRate` to it](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L309-L322).

`exchangeRate` is very important param as it is used in share calculations when user deposits/withdraws. That's why it's important that when vault is ON again after it was OFF, `exchangeRate` is up to date.

In case if vault is off, then exhange rate is not provided to it.
Then when after few rebalancing period, some allocation was provided for the vault and it was marked as ON again and `sendFeedbackToVault` is called, then users can deposit/withdraw again. But the problem is that `exchangeRate` is old there and is likely less than it should be, so depositors receive more shares and withdrawers receive less funds.

Example.
1.At cycle 10 exchangeRate is 1.1.
2.Vault has no more allocations and it's switched off.
3.At cycle 15 new allocations were send to vault. So it's switched on. Current exchangeRate is 1.2.
4.Depositors now can deposit, but `exchangeRate` inside this vault is not fresh, it's from 10th cycle.
5.So some depositors had chance to receive more shares and some withdrawers lost some funds.

I am not sure about severity here, think it's medium because switching on/off can not be often.
## Impact
Incorrect shares calculation
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
When you set vault to on state, you need to send current exchangeRate as well.