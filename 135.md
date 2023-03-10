hyh

high

# There is no price conversion between vault token and provider underlying token amounts in withdrawFromProtocol

## Summary

Vault's withdrawFromProtocol() use its `vaultCurrency` based `_amount` argument without price conversion into Provider's underlying token, which is expected by the calcShares() call. As stablecoin prices can diverge, even massively, it isn't appropriate to assume price equality and use one amount instead of another.

Currently the final withdrawn amount is converted into `vaultCurrency`, but the withdrawal request is being made as if the `vaultCurrency` and Provider's underlying token are worth exactly the same, which almost always isn't the case.

## Vulnerability Detail

Suppose `vaultCurrency` is USDC, while Provider's underlying currency is USDT, Provider is AaveProvider.

Suppose USDT is went through a major regulatory scrutiny and it is now priced at `0.8 USD`, but this is a locally stable situation and the major USDT Vaults now offer quite attractive returns, so Aave USDT was whitelisted.

withdrawFromProtocol() being called with `10^12` amount, which is a request to withdraw `10^6 USDC`, while AaveProvider's calcShares() will be called without price conversion, i.e. it will treat USDC amount supplied as USDT amount as it's the underlying token. Upon withdrawal the result will be converted in `vaultCurrency` USDC, always resulting in 20% less value than it was requested.

## Impact

Withdrawal from a protocol is a base functionality of the Vault and it will malfunction whenever vault currency and Provider's underlying currency aren't equally priced, which is the case along with market volatility.

Net impact is ongoing Vault misbalancing, mostly mild, but sometimes substantial, which will alter the effective distribution vs desired and shift actual results from expected, leading to losses for protocol depositors.

## Code Snippet

withdrawFromProtocol's input argument `_amount` is in `vaultCurrency`, and it is provided after decimals rebase, but without any cross-price conversion to calcShares():

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L303-L316

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
```

All calcShares() methods expect `_amount` to be in the Provider underlying token, not in `vaultCurrency`:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/AaveProvider.sol#L88-L96

```solidity
  /// @notice Calculates how many shares are equal to the amount
  /// @dev Aave exchangeRate is 1
  /// @param _amount Amount in underyling token e.g USDC
  /// @param _aToken Address of protocol LP Token eg aUSDC
  /// @return number of shares i.e LP tokens
  function calcShares(uint256 _amount, address _aToken) external view override returns (uint256) {
    uint256 shares = _amount / exchangeRate(_aToken);
    return shares;
  }
```

...

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L102-L111

```solidity
  /// @notice Calculates how many shares are equal to the amount
  /// @dev returned price from compound is scaled https://compound.finance/docs/ctokens#exchange-rate
  /// @param _amount Amount in underyling token e.g USDC
  /// @param _cToken Address of protocol LP Token eg cUSDC
  /// @return number of shares i.e LP tokens
  function calcShares(uint256 _amount, address _cToken) external view override returns (uint256) {
    uint256 decimals = IERC20Metadata(ICToken(_cToken).underlying()).decimals();
    uint256 shares = (_amount * (10 ** (10 + decimals))) / exchangeRate(_cToken);
    return shares;
  }
```

Cross-currency prices can vary even in the stablecoins case and whenever rebalancing coincides with cross-rate volatility spike the effective Vault weights will be disturbed.

## Tool used

Manual Review

## Recommendation

Consider correcting for the current vault currency to underlying token exchange rate, for example:

```solidity
IController.UniswapParams uniParams = controller.getUniswapParams();
IController.UniswapParams curveParams = controller.getCurveParams();
underlyingAmount = Swap.amountOutMultiSwap(
    Swap.SwapInOut(_amount, address(vaultCurrency), protocol.underlying),
    uniParams.quoter,
    0
);
underlyingAmount = (underlyingAmount * (10000 + curveParams.poolFee)) / 10000;
```

and using the `underlyingAmount` for `shares` estimation via calcShares().