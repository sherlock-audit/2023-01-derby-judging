Jeiwan

high

# Rebalancing can be indefinitely blocked due to ever-increasing `totalWithdrawalRequests`, causing locking of funds in vaults

## Summary
Rebalancing can get stuck indefinitely at the `pushVaultAmounts` step due to an error in the accounting of `totalWithdrawalRequests`. As a result, funds will be locked in vaults since requested withdrawals are only executed after a next successful rebalance.
## Vulnerability Detail
Funds deposited to underlying protocols can only be withdrawn from vaults after a next successful rebalance:
1. a depositor has to make a withdrawal request first, which is [tracked in the current rebalance period](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L160);
1. requested funds can be withdrawn [in the next rebalance period](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L169).

Thus, it's critical that rebalancing doesn't get stuck during one of its stages.

During rebalancing, vaults report their balances to `XChainController` via the [pushTotalUnderlyingToController](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L249) function: the functions sends the [current unlocked (i.e. excluding reserved funds) underlying token balance of the vault](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L253) and the [total amount of withdrawn requests in the current period](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L260). The latter amount is stored in the `totalWithdrawalRequests` storage variable:
1. the variable is [increased when a new withdrawal request is made](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L161);
1. and it's set to 0 [after the vault has been rebalanced](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L338)–it's value is [added to the reserved funds](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L337).

The logic of `totalWithdrawalRequests` is that it tracks only the requested withdrawal amounts in the current period–this amount becomes reserved during rebalancing and is added to `reservedFunds` after the vault has been rebalanced.

When `XChainController` receives underlying balances and withdrawal requests from vaults, it [tracks them internally](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L286-L287). The amounts then used to [calculate how much tokens a vault needs to send or receive after a rebalancing](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L302-L303): the total withdrawal amount is subtracted from vault's underlying balance so that it's excluded from the amounts that will be sent to the protocols and so that it could then be added to the reserved funds of the vault.

However, `totalWithdrawalRequests` in `XChainController` is not reset between rebalancings: when a new rebalancing starts, `XChainController` [receives allocations from the Game](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L203) and calls [resetVaultUnderlying](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L179), which resets the underlying balances receive from vaults in the previous rebalancing. `resetVaultUnderlying` doesn't set `totalWithdrawalRequests` to 0:
```solidity
function resetVaultUnderlying(uint256 _vaultNumber) internal {
  vaults[_vaultNumber].totalUnderlying = 0;
  vaultStage[_vaultNumber].underlyingReceived = 0;
  vaults[_vaultNumber].totalSupply = 0;
}
```

This cause the value of `totalWithdrawalRequests` to accumulate over time. At some point, the total historical amount of all withdrawal requests (which `totalWithdrawalRequests` actually tracks) will be greater than the underlying balance of a vault, and [this line](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L303) will revert due to an underflow in the subtraction:
```solidity
uint256 totalUnderlying = getTotalUnderlyingVault(_vaultNumber) - totalWithdrawalRequests;
```
## Impact
Due to accumulation of withdrawal request amounts in the `totalWithdrawalRequests` variable, `XChainController.pushVaultAmounts` can be blocked indefinitely after the value of `totalWithdrawalRequests` has grown bigger than the value of `totalUnderlying` of a vault. Since withdrawals from vaults are delayed and enable in a next rebalancing period, depositors may not be able to withdraw their funds from vaults, due to a block rebalancing.

While `XChainController` implements a bunch of functions restricted to the guardian that allow the guardian to push a rebalancing through, neither of these functions resets the value of `totalWithdrawalRequests`. If `totalWithdrawalRequests` becomes bigger than `totalUnderlying`, the guardian won't be able to fix the state of `XChainController` and push the rebalancing through.
## Code Snippet
1. When a rebalancing starts, previously reported underlying balances are reset:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L212
1. However, `totalWithdrawalRequests` is never reset:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L180-L182
1. When `XChainController` calculates new amounts of underlying tokens, it subtracts `totalWithdrawalRequests` from `totalUnderlying` to reserve requested amounts in the current period:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L303
1. `totalWithdrawalRequests` is always increased when a vault reports its balances:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L287
## Tool used
Manual Review
## Recommendation
In `XChainController.resetVaultUnderlying`, consider setting `vaults[_vaultNumber].totalWithdrawalRequests` to 0. `totalWithdrawalRequests`, like its `MainVault.totalWithdrawalRequests` counterpart, tracks withdrawal requests only in the current period and should be reset to 0 between rebalancings.