hyh

high

# Vault's savedTotalUnderlying tracks withdrawn funds incorrectly

## Summary

Vault's pullFunds() updates total underlying funds outside Vault variable, `savedTotalUnderlying`, preliminary with the amount requested, not with the amount obtained.

## Vulnerability Detail

pullFunds() reduces `savedTotalUnderlying` before calling for protocol withdrawal, with amount requested for withdrawal, not with amount that was actually withdrawn. Also, when `amountToWithdraw < minimumPull` the `minimumPull` is removed from `savedTotalUnderlying` despite no withdrawal is made in this case.

As amount requested tends to be bigger than amount withdrawn, the result is `savedTotalUnderlying` being understated this way. This effect will accumulate over time.

## Impact

Net impact is ongoing Vault misbalancing by artificially reducing allocations to the underlying DeFi pools as understated `savedTotalUnderlying` means less funds are deemed to be available for each Provider.

This will reduce the effective investment sizes vs desired and diminish the results realized as some funds will remain systemically dormant (not used as calcUnderlyingIncBalance() is reported lower than real).

This is the protocol wide loss for the depositors.

## Code Snippet

`savedTotalUnderlying -= amountToWithdraw` is performed before withdrawFromProtocol() was called:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L107-L127

```solidity
  /// @notice Withdraw from protocols on shortage in Vault
  /// @dev Keeps on withdrawing until the Vault balance > _value
  /// @param _value The total value of vaultCurrency an user is trying to withdraw.
  /// @param _value The (value - current underlying value of this vault) is withdrawn from the underlying protocols.
  function pullFunds(uint256 _value) internal {
    uint256 latestID = controller.latestProtocolId(vaultNumber);
    for (uint i = 0; i < latestID; i++) {
      if (currentAllocations[i] == 0) continue;

      uint256 shortage = _value - vaultCurrency.balanceOf(address(this));
      uint256 balanceProtocol = balanceUnderlying(i);

      uint256 amountToWithdraw = shortage > balanceProtocol ? balanceProtocol : shortage;
>>    savedTotalUnderlying -= amountToWithdraw;

      if (amountToWithdraw < minimumPull) break;
      withdrawFromProtocol(i, amountToWithdraw);

      if (_value <= vaultCurrency.balanceOf(address(this))) break;
    }
  }
```

But actual amount withdrawn from protocol differs from the amount requested:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L307-L336

```solidity
  function withdrawFromProtocol(uint256 _protocolNum, uint256 _amount) internal {
    if (_amount <= 0) return;
    IController.ProtocolInfoS memory protocol = controller.getProtocolInfo(
      vaultNumber,
      _protocolNum
    );

    _amount = (_amount * protocol.uScale) / uScale;
    uint256 shares = IProvider(protocol.provider).calcShares(_amount, protocol.LPToken);
    uint256 balance = IProvider(protocol.provider).balance(address(this), protocol.LPToken);

    if (shares == 0) return;
>>  if (balance < shares) shares = balance;

    IERC20(protocol.LPToken).safeIncreaseAllowance(protocol.provider, shares);
>>  uint256 amountReceived = IProvider(protocol.provider).withdraw(
      shares,
      protocol.LPToken,
      protocol.underlying
    );

    if (protocol.underlying != address(vaultCurrency)) {
>>    _amount = Swap.swapStableCoins(
        Swap.SwapInOut(amountReceived, protocol.underlying, address(vaultCurrency)),
        controller.underlyingUScale(protocol.underlying),
        uScale,
        controller.getCurveParams(protocol.underlying, address(vaultCurrency))
      );
    }
  }
```

This happens because of:

1) shares balance restriction
2) underlying funds availability within Provider
3) swapping to `vaultCurrency` is market dependent and is not deterministic

As this is substantial degree of randomness it can be assumed that withdrawFromProtocol() typically returns somewhat different amount than it was requested.

`savedTotalUnderlying` is then used by calcUnderlyingIncBalance():

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L156-L164

```solidity
  /// @notice Helper to return underlying balance plus totalUnderlying - liquidty for the vault
  /// @return underlying totalUnderlying - liquidityVault
  function calcUnderlyingIncBalance() internal view returns (uint256) {
>>  uint256 totalUnderlyingInclVaultBalance = savedTotalUnderlying +
      getVaultBalance() -
      reservedFunds;
    uint256 liquidityVault = (totalUnderlyingInclVaultBalance * liquidityPerc) / 100;
    return totalUnderlyingInclVaultBalance - liquidityVault;
  }
```

Which defines the allocation during Vault rebalancing:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L135-L154

```solidity
  function rebalance() external nonReentrant {
    require(state == State.RebalanceVault, stateError);
    require(deltaAllocationsReceived, "!Delta allocations");

    rebalancingPeriod++;

    claimTokens();
    settleDeltaAllocation();

>>  uint256 underlyingIncBalance = calcUnderlyingIncBalance();
    uint256[] memory protocolToDeposit = rebalanceCheckProtocols(underlyingIncBalance);

    executeDeposits(protocolToDeposit);
    setTotalUnderlying();

    if (reservedFunds > vaultCurrency.balanceOf(address(this))) pullFunds(reservedFunds);

    state = State.SendRewardsPerToken;
    deltaAllocationsReceived = false;
  }
```

## Tool used

Manual Review

## Recommendation

Consider returning the amount realized in withdrawFromProtocol(): 

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L303-L336

```diff
  /// @notice Withdraw amount from underlying protocol
  /// @dev shares = amount / PricePerShare
  /// @param _protocolNum Protocol number linked to an underlying protocol e.g compound_usdc_01
  /// @param _amount in VaultCurrency to withdraw
- function withdrawFromProtocol(uint256 _protocolNum, uint256 _amount) internal {
+ function withdrawFromProtocol(uint256 _protocolNum, uint256 _amount) internal returns (uint256 amountReceived) {
    if (_amount <= 0) return;
    IController.ProtocolInfoS memory protocol = controller.getProtocolInfo(
      vaultNumber,
      _protocolNum
    );

    _amount = (_amount * protocol.uScale) / uScale;
    uint256 shares = IProvider(protocol.provider).calcShares(_amount, protocol.LPToken);
    uint256 balance = IProvider(protocol.provider).balance(address(this), protocol.LPToken);

    if (shares == 0) return;
    if (balance < shares) shares = balance;

    IERC20(protocol.LPToken).safeIncreaseAllowance(protocol.provider, shares);
-   uint256 amountReceived = IProvider(protocol.provider).withdraw(
+   amountReceived = IProvider(protocol.provider).withdraw(
      shares,
      protocol.LPToken,
      protocol.underlying
    );

    if (protocol.underlying != address(vaultCurrency)) {
-     _amount = Swap.swapStableCoins(
+     amountReceived = Swap.swapStableCoins(
        Swap.SwapInOut(amountReceived, protocol.underlying, address(vaultCurrency)),
        controller.underlyingUScale(protocol.underlying),
        uScale,
        controller.getCurveParams(protocol.underlying, address(vaultCurrency))
      );
    }
  }
```

And updating `savedTotalUnderlying` after the fact with the actual amount in pullFunds():

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L107-L127

```diff
  /// @notice Withdraw from protocols on shortage in Vault
  /// @dev Keeps on withdrawing until the Vault balance > _value
  /// @param _value The total value of vaultCurrency an user is trying to withdraw.
  /// @param _value The (value - current underlying value of this vault) is withdrawn from the underlying protocols.
  function pullFunds(uint256 _value) internal {
    uint256 latestID = controller.latestProtocolId(vaultNumber);
    for (uint i = 0; i < latestID; i++) {
      if (currentAllocations[i] == 0) continue;

      uint256 shortage = _value - vaultCurrency.balanceOf(address(this));
      uint256 balanceProtocol = balanceUnderlying(i);

      uint256 amountToWithdraw = shortage > balanceProtocol ? balanceProtocol : shortage;
-     savedTotalUnderlying -= amountToWithdraw;

      if (amountToWithdraw < minimumPull) break;
-     withdrawFromProtocol(i, amountToWithdraw);
+     uint256 amountWithdrawn = withdrawFromProtocol(i, amountToWithdraw);
+     savedTotalUnderlying -= amountWithdrawn;

      if (_value <= vaultCurrency.balanceOf(address(this))) break;
    }
  }
```

This will also remove `minimumPull` bias that is currently added when `if (amountToWithdraw < minimumPull) break` triggers (no funds are pulled, but `savedTotalUnderlying` is reduced by `minimumPull`).