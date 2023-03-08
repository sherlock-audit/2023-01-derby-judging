Bnke0x0

high

# ERC20 return values not checked

## Summary

## Vulnerability Detail
Some tokens (like USDT) don't correctly implement the EIP20 standard and their transfer/transferFrom function return void instead of a successful boolean. Calling these functions with the correct EIP20 function signatures will always revert.
## Impact
Tokens that don't correctly implement the latest EIP20 spec, like USDT, will be unusable in the protocol as they revert the transaction because of the missing return value. As there is a xToken with USDT as the underlying issue directly applies to the protocol.
## Code Snippet


https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L147

```solidity
IERC20(_token).transferFrom(msg.sender, address(this), _amount);
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L329

```solidity
IERC20(_asset).transferFrom(msg.sender, xController, _amount);
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L372

```solidity
IERC20(_asset).transferFrom(msg.sender, _vault, _amount);
```
## Tool used

Manual Review

## Recommendation
We recommend using OpenZeppelin’s SafeERC20 versions with the safeTransfer and safeTransferFrom functions that handle the return value check as well as non-standard-compliant tokens.