csanuragjain

medium

# Max cap on gov fee is missing

## Summary
If gov fee is set too high then User may lose funds as most of fund will go as fees

## Vulnerability Detail
1. Lets say gov fee is set to 100%
2. `withdrawAllowance` is called to settle the position
3. This internally calls `transferFunds`

```solidity
function transferFunds(address _receiver, uint256 _value) internal {
    uint256 govFee = (_value * governanceFee) / 10_000;

    vaultCurrency.safeTransfer(getDao(), govFee);
    vaultCurrency.safeTransfer(_receiver, _value - govFee);
  }
```

4. Since gov fee is 100% so full user amount is taken as fee and sent to DAO

## Impact
User may lose funds

## Code Snippet
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L458

## Tool used
Manual Review

## Recommendation
Keep a max cap for gov fees