hyh

medium

# Vault can lose rewards due to lack of precision

## Summary

Vault's storePriceAndRewards() can accrue no rewards as precision can be lost when price() has low decimals.

For example, price() has 6 decimals for Idle USDC and USDT strategies.

## Vulnerability Detail

Suppose storePriceAndRewards() is called for Idle USDC protocol, this USDC Vault is relatively new and `Totalunderlying = TotalUnderlyingInProtocols - BalanceVault = USD 30k`, performance fee is `5%` and it's `1 mln` Derby tokens staked, i.e. `totalAllocatedTokens = 1_000_000 * 1e18`, while `lastPrices[_protocolId] = 1_100_000` (which is `1.1 USDC`, Idle has price scaled by underlying decimals and tracks accumulated strategy share price).

Let's say this provider shows stable returns with `1.7% APY`, suppose market rates are low and this is somewhat above market. In bi-weekly terms it can correspond to `priceDiff = 730`, as, roughly excluding price appreciation that doesn't influence much here, `(730.0 / 1100000 + 1)**26 - 1 = 1.7%`.

In this case `rewardPerLockedToken` will be `30000 * 10**6 * 5 * 730 / (1_000_000 * 1_100_000 * 100) = 0` for all rebalancing periods, i.e. no rewards at all will be allocated for the protocol despite it having positive performance.

## Impact

Rewards can be lost for traders who allocated to protocols where price() has low decimals.

## Code Snippet

Vault's storePriceAndRewards() calculates strategy performance reward as `nominator / denominator`, where `nominator = _totalUnderlying * performanceFee * priceDiff`:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L220-L245

```solidity
  /// @notice Stores the historical price and the reward per rounded locked token, ignoring decimals.
  /// @dev formula yield protocol i at time t: y(it) = (P(it) - P(it-1)) / P(it-1).
  /// @dev formula rewardPerLockedToken for protocol i at time t: r(it) = y(it) * TVL(t) * perfFee(t) / totalLockedTokens(t)
  /// @dev later, when the total rewards are calculated for a game player we multiply this (r(it)) by the locked tokens on protocol i at time t
  /// @param _totalUnderlying Totalunderlying = TotalUnderlyingInProtocols - BalanceVault.
  /// @param _protocolId Protocol id number.
  function storePriceAndRewards(uint256 _totalUnderlying, uint256 _protocolId) internal {
    uint256 currentPrice = price(_protocolId);
    if (lastPrices[_protocolId] == 0) {
      lastPrices[_protocolId] = currentPrice;
      return;
    }

    int256 priceDiff = int256(currentPrice - lastPrices[_protocolId]);
    int256 nominator = (int256(_totalUnderlying * performanceFee) * priceDiff);
    int256 totalAllocatedTokensRounded = totalAllocatedTokens / 1E18;
    int256 denominator = totalAllocatedTokensRounded * int256(lastPrices[_protocolId]) * 100; // * 100 cause perfFee is in percentages

    if (totalAllocatedTokensRounded == 0) {
      rewardPerLockedToken[rebalancingPeriod][_protocolId] = 0;
    } else {
      rewardPerLockedToken[rebalancingPeriod][_protocolId] = nominator / denominator;
    }

    lastPrices[_protocolId] = currentPrice;
  }
```

price() is Provider's exchangeRate():

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L381-L390

```solidity
  /// @notice Get price for underlying protocol
  /// @param _protocolNum Protocol number linked to an underlying protocol e.g compound_usdc_01
  /// @return protocolPrice Price per lp token
  function price(uint256 _protocolNum) public view returns (uint256) {
    IController.ProtocolInfoS memory protocol = controller.getProtocolInfo(
      vaultNumber,
      _protocolNum
    );
    return IProvider(protocol.provider).exchangeRate(protocol.LPToken);
  }
```

IdleProvider's exchangeRate() is scaled with underlying token decimals:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/IdleProvider.sol#L113-L118

```solidity
  /// @notice Exchange rate of underyling protocol token
  /// @param _iToken Address of protocol LP Token eg yUSDC
  /// @return price of LP token
  function exchangeRate(address _iToken) public view override returns (uint256) {
    return IIdle(_iToken).tokenPrice();
  }
```

https://github.com/Idle-Labs/idle-contracts/blob/develop/contracts/IdleTokenV3_1.sol#L240-L245

```solidity
  /**
   * IdleToken price calculation, in underlying
   *
   * @return : price in underlying token
   */
  function tokenPrice() external view returns (uint256) {}
```

## Tool used

Manual Review

## Recommendation

Consider enhancing performance of the rewards calculation and Game's `baskets[_basketId].totalUnRedeemedRewards`, for example:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L46-L47

```solidity
  uint256 public uScale;

  uint256 public minimumPull;

+ uint256 public BASE_SCALE = 1e18;
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L220-L245

```solidity
  /// @notice Stores the historical price and the reward per rounded locked token, ignoring decimals.
  /// @dev formula yield protocol i at time t: y(it) = (P(it) - P(it-1)) / P(it-1).
  /// @dev formula rewardPerLockedToken for protocol i at time t: r(it) = y(it) * TVL(t) * perfFee(t) / totalLockedTokens(t)
  /// @dev later, when the total rewards are calculated for a game player we multiply this (r(it)) by the locked tokens on protocol i at time t
  /// @param _totalUnderlying Totalunderlying = TotalUnderlyingInProtocols - BalanceVault.
  /// @param _protocolId Protocol id number.
  function storePriceAndRewards(uint256 _totalUnderlying, uint256 _protocolId) internal {
    uint256 currentPrice = price(_protocolId);
    if (lastPrices[_protocolId] == 0) {
      lastPrices[_protocolId] = currentPrice;
      return;
    }

    int256 priceDiff = int256(currentPrice - lastPrices[_protocolId]);
-   int256 nominator = (int256(_totalUnderlying * performanceFee) * priceDiff);
+   int256 nominator = (int256(_totalUnderlying * performanceFee) * priceDiff * BASE_SCALE);
    int256 totalAllocatedTokensRounded = totalAllocatedTokens / 1E18;
    int256 denominator = totalAllocatedTokensRounded * int256(lastPrices[_protocolId]) * 100; // * 100 cause perfFee is in percentages

    if (totalAllocatedTokensRounded == 0) {
      rewardPerLockedToken[rebalancingPeriod][_protocolId] = 0;
    } else {
      rewardPerLockedToken[rebalancingPeriod][_protocolId] = nominator / denominator;
    }

    lastPrices[_protocolId] = currentPrice;
  }
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L69-L70

```solidity
  // percentage of tokens that will be sold at negative rewards
  uint256 internal negativeRewardFactor;

+ uint256 public BASE_SCALE = 1e18;
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L296-L306

```solidity
  function redeemNegativeRewards(
    uint256 _basketId,
    uint256 _unlockedTokens
  ) internal returns (uint256) {
    int256 unredeemedRewards = baskets[_basketId].totalUnRedeemedRewards;
    if (unredeemedRewards > negativeRewardThreshold) return 0;

    uint256 tokensToBurn = (uint(-unredeemedRewards) * negativeRewardFactor) / 100;
    tokensToBurn = tokensToBurn < _unlockedTokens ? tokensToBurn : _unlockedTokens;

-   baskets[_basketId].totalUnRedeemedRewards += int((tokensToBurn * 100) / negativeRewardFactor);
+   baskets[_basketId].totalUnRedeemedRewards += int((tokensToBurn * 100 * BASE_SCALE) / negativeRewardFactor);
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L542-L553

```solidity
  /// @notice redeem funds from basket in the game.
  /// @dev makes a call to the vault to make the actual transfer because the vault holds the funds.
  /// @param _basketId Basket ID (tokenID) in the BasketToken (NFT) contract.
  function redeemRewards(uint256 _basketId) external onlyBasketOwner(_basketId) {
-   int256 amount = baskets[_basketId].totalUnRedeemedRewards;
+   int256 amount = baskets[_basketId].totalUnRedeemedRewards / BASE_SCALE;
    require(amount > 0, "Nothing to claim");

    baskets[_basketId].totalRedeemedRewards += amount;
    baskets[_basketId].totalUnRedeemedRewards = 0;

    IVault(homeVault).redeemRewardsGame(uint256(amount), msg.sender);
  }
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L19-L32

```solidity
  struct Basket {
    // the vault number for which this Basket was created
    uint256 vaultNumber;
    // last period when this Basket got rebalanced
    uint256 lastRebalancingPeriod;
    // nr of total allocated tokens
    int256 nrOfAllocatedTokens;
    // total build up rewards
-   int256 totalUnRedeemedRewards;
+   int256 totalUnRedeemedRewards;  // in {underlying + 18} decimals precision
    // total redeemed rewards
    int256 totalRedeemedRewards;
    // (basket => vaultNumber => chainId => allocation)
    mapping(uint256 => mapping(uint256 => int256)) allocations;
  }
```