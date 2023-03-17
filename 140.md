Ch_301

high

# The protocol could not handle multiple vaults correctly

## Summary
The protocol needs to handle multiple vaults correctly.
If there are three vaults (e.g.USDC, USDT, DAI) the protocol needs to rebalance them all without any problems

## Vulnerability Detail
The protocol needs to invoke [pushAllocationsToController()](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L424-L445) every `rebalanceInterval` to push **totalDeltaAllocations** from **Game** to **xChainController**.

`pushAllocationsToController()` invoke `rebalanceNeeded()` to check if a rebalance is needed based on the set interval
and it uses the state variable `lastTimeStamp` to do the calculations  

```solidity
  function rebalanceNeeded() public view returns (bool) {
    return (block.timestamp - lastTimeStamp) > rebalanceInterval || msg.sender == guardian;
  }
```

But in the first invoking (for USDC vault) of `pushAllocationsToController()`  it will update the state variable `lastTimeStamp` to the current `block.timestamp`

```solidity
lastTimeStamp = block.timestamp;
```

Now when you invoke (for DAI vault) `pushAllocationsToController()`. It will revert because of  
```solidity
require(rebalanceNeeded(), "No rebalance needed");
```
So if the protocol has two vaults or more (USDC, USDT, DAI) you can only do one rebalance every `rebalanceInterval`

## Impact
- The protocol could not handle multiple vaults correctly
- Both Users and Game players will lose funds because the MainVault will not rebalance the protocols at the right time with the right values  

## Code Snippet
```solidity
  function pushAllocationsToController(uint256 _vaultNumber) external payable {
    require(rebalanceNeeded(), "No rebalance needed");
    for (uint k = 0; k < chainIds.length; k++) {
      require(
        getVaultAddress(_vaultNumber, chainIds[k]) != address(0),
        "Game: not a valid vaultnumber"
      );
      require(
        !isXChainRebalancing[_vaultNumber][chainIds[k]],
        "Game: vault is already rebalancing"
      );
      isXChainRebalancing[_vaultNumber][chainIds[k]] = true;
    }
```
## Tool used

Manual Review

## Recommendation
Keep tracking the `lastTimeStamp` for every `_vaultNumber` by using an array 

