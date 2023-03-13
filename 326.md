Jeiwan

medium

# An inactive vault can disrupt rebalancing of active vaults

## Summary
An inactive vault can send its total underlying amount to the `XChainController` and disrupt rebalancing of active vaults by increasing the `underlyingReceived` counter:
1. if `pushVaultAmounts` is called before `underlyingReceived` overflows, rebalancing of one of the active vault may get stuck since the vault won't receive XChain allocations;
1. if `pushVaultAmounts` after all active vaults and at least one inactive vault has reported their underlying amounts, rebalancing of all vaults will get stuck.
## Vulnerability Detail
Rebalancing of vaults starts when [Game.pushAllocationsToController](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L424) is called. The function sends the allocations made by gamers to the `XChainController`. `XChainController` receives them in the [receiveAllocationsFromGame](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L193) function. In the [settleCurrentAllocation](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L224) function, a vault is marked as inactive if it has no allocations and there are no new allocations for the vault. `receiveAllocationsFromGameInt` [remembers](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L213) the [number of active vaults](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L208).

The next step of the rebalancing process is reporting vault underlying token balances to the `XChainController` by calling [MainVault.pushTotalUnderlyingToController](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L249). As you can see, the function can be called in an inactive vault (the only modifier of the function, `onlyWhenIdle`, doesn't check that `vaultOff` is `false`). `XChainController` receives underlying balances in the [setTotalUnderlying](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L258) function: notice that the function [increases the number of balances it has received](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L288).

Next step is the [XChainController.pushVaultAmounts](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L295) function, which calculates how much tokens each vault should receive after gamers have changed their allocations. The function can be called only [when all active vaults have reported their underlying balances](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L298):
```solidity
modifier onlyWhenUnderlyingsReceived(uint256 _vaultNumber) {
  require(
    vaultStage[_vaultNumber].underlyingReceived == vaultStage[_vaultNumber].activeVaults,
    "Not all underlyings received"
  );
  _;
}
```

However, as we saw above, inactive vaults can also report their underlying balances and increase the `underlyingReceived` counter–if this is abused mistakenly or intentionally (e.g. by a malicious actor), vaults may end up in a corrupted state. Since all the functions involved in rebalancing are not restricted (including `pushTotalUnderlyingToController` and `pushVaultAmounts`), a malicious actor can intentionally disrupt accounting of vaults or block a rebalancing.
## Impact
1. If an inactive vault reports its underlying balances instead of an active vault (i.e. `pushVaultAmounts` is called when `underlyingReceived` is equal `activeVaults`), the active vault will be excluded from rebalancing and it won't receive updated allocations in the current period. Since the rebalancing interval is 2 weeks, the vault will lose the increased yield that might've been generated thanks to new allocations.
1. If an inactive vault reports its underlying balances in addition to all active vaults (i.e. `pushVaultAmounts` is called when `underlyingReceived` is greater than `activeVaults`), then `pushVaultAmounts` will always revert and rebalancing will get stuck.
## Code Snippet
1. An inactive vault can send its underlying balance to the `XChainController`:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L249
1. The `XChainController` can receive underlying balances from inactive vaults:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L264
1. `underlyingReceived` is increased when underlying balances are received:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L288
1. `pushVaultAmounts` can only be executed when the number of vaults that have reported their balances equals the number of active vaults:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L298
## Tool used
Manual Review
## Recommendation
In the `MainVault.pushTotalUnderlyingToController` function, consider disallowing inactive vaults (vaults that have `vaultOff` set to `true`) report their underlying balances.