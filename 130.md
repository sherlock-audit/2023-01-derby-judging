rvierdiiev

medium

# Vault.blacklistProtocol doesn't claim rewards

## Summary
Vault.blacklistProtocol doesn't claim rewards
## Vulnerability Detail
`Vault.blacklistProtocol` is created for emergency cases and it's needed to remove protocol from protocols that allowed to deposit.
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L477-L483
```solidity
  function blacklistProtocol(uint256 _protocolNum) external onlyGuardian {
    uint256 balanceProtocol = balanceUnderlying(_protocolNum);
    currentAllocations[_protocolNum] = 0;
    controller.setProtocolBlacklist(vaultNumber, _protocolNum);
    savedTotalUnderlying -= balanceProtocol;
    withdrawFromProtocol(_protocolNum, balanceProtocol);
  }
```

Once protocol is blacklisted, protocol has some time to call `claimTokens` function as it will be [not possible to claim rewards](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L407) for it on next cycle as allocation will be 0.

That's why i believe that token should be claimed inside `blacklistProtocol` function.
## Impact
Rewards will be not claimed
## Code Snippet
Provided above
## Tool used

Manual Review

## Recommendation
Add claiming to the `blacklistProtocol` function.