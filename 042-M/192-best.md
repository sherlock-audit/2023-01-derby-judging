Ch_301

high

# Malicious users could set allocations to a blacklist Protocol and break the rebalancing logic

## Summary
`game.sol` pushes `deltaAllocations` to vaults by [pushAllocationsToVaults()](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L465-L477) and it deletes all the value of the `deltas`

```solidity
vaults[_vaultNumber].deltaAllocationProtocol[_chainId][i] = 0;
```

## Vulnerability Detail
Malicious users could set allocations to a blacklist Protocol. 
If only one of the `Baskets` has a non-zero value to a **Protocol on blacklist**
[receiveProtocolAllocations()](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L343-L345) will revert 
`receiveProtocolAllocations().receiveProtocolAllocationsInt().setDeltaAllocationsInt()`
```solidity
  function setDeltaAllocationsInt(uint256 _protocolNum, int256 _allocation) internal {
    require(!controller.getProtocolBlacklist(vaultNumber, _protocolNum), "Protocol on blacklist");
    deltaAllocations[_protocolNum] += _allocation;
    deltaAllocatedTokens += _allocation;
  }
```
and You won't be able to execute [rebalance()](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L135-L154) 


## Impact
The guardian isn't able to restart the protocol manually. 
`game.sol` loses the value of the `deltas`.
The whole system is down.

## Code Snippet

## Tool used

Manual Review

## Recommendation
You should check if the Protocol on the blacklist when Game players `rebalanceBasket()`