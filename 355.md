hyh

high

# YearnProvider freezes yearn tokens on partial withdrawal

## Summary

YearnProvider's withdraw() doesn't account for partial withdrawal situation, which isn't rare, so the unused part of user shares end up being stuck on the contract balance as there is no mechanics to retrieve them thereafter.

## Vulnerability Detail

Full amount of the shares user requested to be burned is transferred to YearnProvider, but only part of it can be utilized by Yearn withdrawal.

Liquidity shortage (squeeze) is common enough situation, for example it can occur whenever part of the Yearn strategy is tied to a lending market that have high utilization at the moment of the call.

## Impact

Part of protocol funds can be permanently frozen on YearnProvider contract balance (as it's not operating the funds itself, always referencing the caller Vault).

As Provider's withdraw is routinely called by Vault managing the aggregated funds distribution, the freeze amount can be massive enough and will be translated to a loss for many users.

## Code Snippet

withdraw() transfers all the requested `_amount` from the user, but do not return the remainder `_yToken` amount:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/YearnProvider.sol#L44-L66

```solidity
  function withdraw(
    uint256 _amount,
    address _yToken,
    address _uToken
  ) external override returns (uint256) {
    uint256 balanceBefore = IERC20(_uToken).balanceOf(msg.sender);

    require(
      IYearn(_yToken).transferFrom(msg.sender, address(this), _amount) == true,
      "Error transferFrom"
    );

    uint256 uAmountReceived = IYearn(_yToken).withdraw(_amount);
    IERC20(_uToken).safeTransfer(msg.sender, uAmountReceived);

    uint256 balanceAfter = IERC20(_uToken).balanceOf(msg.sender);
    require(
      (balanceAfter - balanceBefore - uAmountReceived) == 0,
      "Error Withdraw: under/overflow"
    );

    return uAmountReceived;
  }
```

`uAmountReceived` can correspond only to a part of shares `_amount` obtained from the caller.

Yearn withdrawal is not guaranteed to be full, `value` returned and `shares` burned depend on availability (i.e. `shares < maxShares` is valid case):

https://github.com/yearn/yearn-vaults/blob/master/contracts/Vault.vy#L1144-L1167

```vyper
        # NOTE: We have withdrawn everything possible out of the withdrawal queue
        #       but we still don't have enough to fully pay them back, so adjust
        #       to the total amount we've freed up through forced withdrawals
        if value > vault_balance:
            value = vault_balance
            # NOTE: Burn # of shares that corresponds to what Vault has on-hand,
            #       including the losses that were incurred above during withdrawals
            shares = self._sharesForAmount(value + totalLoss)

        # NOTE: This loss protection is put in place to revert if losses from
        #       withdrawing are more than what is considered acceptable.
        assert totalLoss <= maxLoss * (value + totalLoss) / MAX_BPS

    # Burn shares (full value of what is being withdrawn)
    self.totalSupply -= shares
    self.balanceOf[msg.sender] -= shares
    log Transfer(msg.sender, ZERO_ADDRESS, shares)
    
    self.totalIdle -= value
    # Withdraw remaining balance to _recipient (may be different to msg.sender) (minus fee)
    self.erc20_safe_transfer(self.token.address, recipient, value)
    log Withdraw(recipient, shares, value)
    
    return value
```

The remaining part of the shares, `maxShares - shares`, end up left on the YearnProvider balance.

Provider's withdraw() is used by the Vault, where `amountReceived` is deemed corresponding to the full `shares` spent:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L303-L326

```solidity
  /// @notice Withdraw amount from underlying protocol
  /// @dev shares = amount / PricePerShare
  /// @param _protocolNum Protocol number linked to an underlying protocol e.g compound_usdc_01
  /// @param _amount in VaultCurrency to withdraw
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
    if (balance < shares) shares = balance;

    IERC20(protocol.LPToken).safeIncreaseAllowance(protocol.provider, shares);
    uint256 amountReceived = IProvider(protocol.provider).withdraw(
      shares,
      protocol.LPToken,
      protocol.underlying
    );
```

## Tool used

Manual Review

## Recommendation

Consider returning the unused shares to the caller:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/YearnProvider.sol#L44-L66

```diff
  function withdraw(
    uint256 _amount,
    address _yToken,
    address _uToken
  ) external override returns (uint256) {
    uint256 balanceBefore = IERC20(_uToken).balanceOf(msg.sender);
+   uint256 sharesBefore = IERC20(_yToken).balanceOf(address(this));

    require(
      IYearn(_yToken).transferFrom(msg.sender, address(this), _amount) == true,
      "Error transferFrom"
    );

    uint256 uAmountReceived = IYearn(_yToken).withdraw(_amount);
    IERC20(_uToken).safeTransfer(msg.sender, uAmountReceived);

    uint256 balanceAfter = IERC20(_uToken).balanceOf(msg.sender);
    require(
      (balanceAfter - balanceBefore - uAmountReceived) == 0,
      "Error Withdraw: under/overflow"
    );
+   uint256 sharesAfter = IERC20(_yToken).balanceOf(address(this)); 
+   if (sharesAfter > sharesBefore) {
+       IERC20(_yToken).safeTransfer(msg.sender, sharesAfter - sharesBefore);
+   }

    return uAmountReceived;
  }
```