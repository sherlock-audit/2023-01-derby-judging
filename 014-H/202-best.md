hyh

high

# CompoundProvider's balanceUnderlying and calcShares outputs are scaled incorrectly

## Summary

balanceUnderlying() returns values scaled with CToken decimals instead of the underlying decimals.

calcShares() also incorrectly scales its output, which have underlying decimals instead of the CToken decimals.

## Vulnerability Detail

Underlying reason is exchangeRate() output is similarly and incorrectly processed in both functions. This results in wrong decimals of the both return values.

CToken and underlying decimals can differ, for example cDAI has 8 decimals, while DAI has 18.

## Impact

Amount of the underlying held by Compound pool and amount of shares needed are misstated by magnitudes, which can lead to protocol-wide losses.

## Code Snippet

balanceUnderlying() cancels out exchangeRate() decimals with dividing by `10^(18 - 8 + Underlying Token Decimals)`, which results in CToken decimals:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L86-L100

```solidity
  /// @notice Get balance from address in underlying token
  /// @param _address Address to request balance from, most likely a Vault
  /// @param _cToken Address of protocol LP Token eg cUSDC
  /// @return balance in underlying token
  function balanceUnderlying(
    address _address,
    address _cToken
  ) public view override returns (uint256) {
    uint256 balanceShares = balance(_address, _cToken);
    // The returned exchange rate from comp is scaled by 1 * 10^(18 - 8 + Underlying Token Decimals).
    uint256 price = exchangeRate(_cToken);
    uint256 decimals = IERC20Metadata(ICToken(_cToken).underlying()).decimals();

    return (balanceShares * price) / 10 ** (10 + decimals);
  }
```

As balance() returns CToken balance and have CToken decimals:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L113-L120

```solidity
  /// @notice Get balance of cToken from address
  /// @param _address Address to request balance from
  /// @param _cToken Address of protocol LP Token eg cUSDC
  /// @return number of shares i.e LP tokens
  function balance(address _address, address _cToken) public view override returns (uint256) {
    uint256 _balanceShares = ICToken(_cToken).balanceOf(_address);
    return _balanceShares;
  }
```

While exchangeRate() is CToken's exchangeRateStored() with (18 - 8 + Underlying Token Decimals) decimals:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L122-L129

```solidity
  /// @notice Exchange rate of underyling protocol token
  /// @dev returned price from compound is scaled https://compound.finance/docs/ctokens#exchange-rate
  /// @param _cToken Address of protocol LP Token eg cUSDC
  /// @return price of LP token
  function exchangeRate(address _cToken) public view override returns (uint256) {
    uint256 _price = ICToken(_cToken).exchangeRateStored();
    return _price;
  }
```

https://docs.compound.finance/v2/ctokens/#exchange-rate

So balanceUnderlying() have CTokens decimals, which differ from underlying decimals, for example cUSDC have decimals of 8:

https://etherscan.io/token/0x39aa39c021dfbae8fac545936693ac917d5e7563

While USDC have decimals of 6:

https://etherscan.io/token/0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48

cDAI has 8 decimals:

https://etherscan.io/token/0x5d3a536e4d6dbd6114cc1ead35777bab948e3643

DAI has 18:

https://etherscan.io/token/0x6b175474e89094c44da98b954eedeac495271d0f

balanceUnderlying() is used in Vault rebalancing:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L178-L192

```solidity
  function rebalanceCheckProtocols(
    uint256 _newTotalUnderlying
  ) internal returns (uint256[] memory) {
    uint256[] memory protocolToDeposit = new uint[](controller.latestProtocolId(vaultNumber));
    uint256 latestID = controller.latestProtocolId(vaultNumber);
    for (uint i = 0; i < latestID; i++) {
      bool isBlacklisted = controller.getProtocolBlacklist(vaultNumber, i);

      storePriceAndRewards(_newTotalUnderlying, i);

      if (isBlacklisted) continue;
      setAllocation(i);

      int256 amountToProtocol = calcAmountToProtocol(_newTotalUnderlying, i);
      uint256 currentBalance = balanceUnderlying(i);
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L111-L118

```solidity
  function pullFunds(uint256 _value) internal {
    uint256 latestID = controller.latestProtocolId(vaultNumber);
    for (uint i = 0; i < latestID; i++) {
      if (currentAllocations[i] == 0) continue;

      uint256 shortage = _value - vaultCurrency.balanceOf(address(this));
      uint256 balanceProtocol = balanceUnderlying(i);

```

Same for calcShares(), where `shares` scale is corrected wrongly, it has underlying decimals instead of CToken decimals:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L107-L111

```solidity
  function calcShares(uint256 _amount, address _cToken) external view override returns (uint256) {
    uint256 decimals = IERC20Metadata(ICToken(_cToken).underlying()).decimals();
    uint256 shares = (_amount * (10 ** (10 + decimals))) / exchangeRate(_cToken);
    return shares;
  }
```

calcShares() is used in rebalancing as well:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L307-L315

```solidity
  function withdrawFromProtocol(uint256 _protocolNum, uint256 _amount) internal {
    if (_amount <= 0) return;
    IController.ProtocolInfoS memory protocol = controller.getProtocolInfo(
      vaultNumber,
      _protocolNum
    );

    _amount = (_amount * protocol.uScale) / uScale;
    uint256 shares = IProvider(protocol.provider).calcShares(_amount, protocol.LPToken);
```

## Tool used

Manual Review

## Recommendation

Compound exchange rate is specifically scaled so that `CToken_balance * price` has `18 + Underlying Token Decimals`, so it's enough to use `1e18`:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L86-L100

```diff
  /// @notice Get balance from address in underlying token
  /// @param _address Address to request balance from, most likely a Vault
  /// @param _cToken Address of protocol LP Token eg cUSDC
  /// @return balance in underlying token
  function balanceUnderlying(
    address _address,
    address _cToken
  ) public view override returns (uint256) {
    uint256 balanceShares = balance(_address, _cToken);
    // The returned exchange rate from comp is scaled by 1 * 10^(18 - 8 + Underlying Token Decimals).
    uint256 price = exchangeRate(_cToken);
    uint256 decimals = IERC20Metadata(ICToken(_cToken).underlying()).decimals();

-   return (balanceShares * price) / 10 ** (10 + decimals);
+   return (balanceShares * price) / 1e18;
  }
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/CompoundProvider.sol#L102-L111

```diff
  /// @notice Calculates how many shares are equal to the amount
  /// @dev returned price from compound is scaled https://compound.finance/docs/ctokens#exchange-rate
  /// @param _amount Amount in underyling token e.g USDC
  /// @param _cToken Address of protocol LP Token eg cUSDC
  /// @return number of shares i.e LP tokens
  function calcShares(uint256 _amount, address _cToken) external view override returns (uint256) {
-   uint256 decimals = IERC20Metadata(ICToken(_cToken).underlying()).decimals();
-   uint256 shares = (_amount * (10 ** (10 + decimals))) / exchangeRate(_cToken);
+   uint256 shares = _amount * 1e18 / exchangeRate(_cToken);
    return shares;
  }
```