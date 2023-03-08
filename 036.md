SunSec

medium

# Did not Approve to zero first

## Summary

## Vulnerability Detail
Some ERC20 tokens like USDT require resetting the approval to 0 first before being able to reset it to another value.
The ohm.approve, pairToken.approve, pool.approve  function does not do this - unlike OpenZeppelin's safeApprove implementation.
## Impact
unsafe ERC20 approve that do not handle non-standard erc20 behavior.
1.Some token contracts do not return any value.
2.Some token contracts revert the transaction when the allowance is not zero.
## Code Snippet
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L150
```solidity
    // This contract approves transfer to Connext
    IERC20(_token).approve(address(connext), _amount);
```

## Tool used
Manual Review

## Recommendation
It is recommended to set the allowance to zero before increasing the allowance and use safeApprove/safeIncreaseAllowance.