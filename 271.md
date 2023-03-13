immeas

medium

# `XChainController::sendFundsToVault` can be griefed and leave `XChainController` in a bad state

## Summary
A user can grief the send funds to vault state transition during by calling `sendFundsToVault` multiple times with the same vault.

## Vulnerability Detail
During rebalancing, some vaults might need funds sent to them. They will be in state `WaitingForFunds`. To transition from here any user can trigger `XChainController` to send them funds by calling `sendFundsToVault`.

This is trigger per chain and will transfer funds from `XChainController` to the respective vaults on each chain.

At the end, when the vaults on each chain are processed and either have gotten funds sent to them or didn't need to `sendFundsToVaults` will trigger the state for this `vaultNumber` to be reset.

However, when transferring funds, there's never any check that this chain has not already been processed. So any user could simply call this function for a vault that either has no funds to transfer or where there's enough funds in `XChainController` and trigger the state reset for the vault.

PoC in `xChaincontroller.test.ts`, run after `4.5) Trigger vaults to transfer funds to xChainController`:
```javascript
  it('5) Grief xChainController send funds to vaults', async function () {
    await xChainController.sendFundsToVault(vaultNumber, slippage, 10000, 0, { value: 0, });
    await xChainController.sendFundsToVault(vaultNumber, slippage, 10000, 0, { value: 0, });
    await xChainController.sendFundsToVault(vaultNumber, slippage, 10000, 0, { value: 0, });
    await xChainController.sendFundsToVault(vaultNumber, slippage, 10000, 0, { value: 0, });

    expect(await xChainController.getFundsReceivedState(vaultNumber)).to.be.equal(0);

    expect(await vault3.state()).to.be.equal(3);

    // can't trigger state change anymore
    await expect(xChainController.sendFundsToVault(vaultNumber, slippage, 1000, relayerFee, {value: parseEther('0.1'),})).to.be.revertedWith('Not all funds received');
  });
```

## Impact
XChainController ends up out of sync with the vault(s) that were supposed to receive funds.

`guardian` can resolve this by resetting the states using admin functions but these functions can still be frontrun by a malicious user.

Until this is resolved the rebalancing of the impacted vaults cannot continue.

## Code Snippet
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L439

## Tool used
Manual Review, hardhat

## Recommendation
I recommend the protocol either keeps track of which vaults have been sent funds in `XChainController`.

or changes so a vault can only receive funds when waiting for them:
```diff
diff --git a/derby-yield-optimiser/contracts/MainVault.sol b/derby-yield-optimiser/contracts/MainVault.sol
index 8739e24..d475ee6 100644
--- a/derby-yield-optimiser/contracts/MainVault.sol
+++ b/derby-yield-optimiser/contracts/MainVault.sol
@@ -328,7 +328,7 @@ contract MainVault is Vault, VaultToken {
   /// @notice Step 5 end; Push funds from xChainController to vaults
   /// @notice Receiving feedback from xController when funds are received, so the vault can rebalance
   function receiveFunds() external onlyXProvider {
-    if (state != State.WaitingForFunds) return;
+    require(state == State.WaitingForFunds,stateError);
     settleReservedFunds();
   }
 

```