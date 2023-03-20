hyh

medium

# Game's negativeRewardThreshold can be set positive, which will break protocol token unlock logic

## Summary

Setting `negativeRewardThreshold` to a positive value is allowed, but this will cause protocol malfunction, when Vault will be burning all the unlocked tokens of a user on every basket rebalancing whenever `0 < baskets[_basketId].totalUnRedeemedRewards <= negativeRewardThreshold`.

## Vulnerability Detail

`negativeRewardThreshold` is a `int256` that is set without value control, i.e. can be set to positive value.

If that is done by Guardian by operational mistake or with a malicious intent, the Derby token locking/unlocking logic can be burning all unlocked tokens, returning nothing to a user.

## Impact

redeemNegativeRewards() will be burning all the unlocked tokens on each basket rebalance when `0 < baskets[_basketId].totalUnRedeemedRewards <= negativeRewardThreshold`.

## Code Snippet

`negativeRewardThreshold` isn't controlled and can be set to any value:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L626-L630

```solidity
  /// @notice Setter for threshold at which user tokens will be sold / burned
  /// @param _threshold treshold in vaultCurrency e.g USDC, must be negative
  function setNegativeRewardThreshold(int256 _threshold) external onlyDao {
    negativeRewardThreshold = _threshold;
  }
```

If `negativeRewardThreshold > 0` redeemNegativeRewards() can break up system accounting via `uint(-unredeemedRewards)` conversion:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L290-L311

```solidity
  /// @notice IMPORTANT: The negativeRewardFactor takes in account an approximation of the price of derby tokens by the dao
  /// @notice IMPORTANT: This will change to an exact price when there is a derby token liquidity pool
  /// @notice Calculates if there are any negative rewards and how many tokens to burn
  /// @param _basketId Basket ID (tokenID) in the BasketToken (NFT) contract
  /// @param _unlockedTokens Amount of derby tokens to unlock and send to user
  /// @return tokensToBurn Amount of derby tokens that are burned
  function redeemNegativeRewards(
    uint256 _basketId,
    uint256 _unlockedTokens
  ) internal returns (uint256) {
    int256 unredeemedRewards = baskets[_basketId].totalUnRedeemedRewards;
    if (unredeemedRewards > negativeRewardThreshold) return 0;

    uint256 tokensToBurn = (uint(-unredeemedRewards) * negativeRewardFactor) / 100;
    tokensToBurn = tokensToBurn < _unlockedTokens ? tokensToBurn : _unlockedTokens;

    baskets[_basketId].totalUnRedeemedRewards += int((tokensToBurn * 100) / negativeRewardFactor);

    IERC20(derbyToken).safeTransfer(homeVault, tokensToBurn);

    return tokensToBurn;
  }
```

As when `unredeemedRewards` is positive, but `unredeemedRewards <= negativeRewardThreshold`, the `uint(-unredeemedRewards)` will not revert, but be result of explicit conversion, making `tokensToBurn` huge:

https://docs.soliditylang.org/en/latest/types.html#explicit-conversions

So it will be `tokensToBurn = _unlockedTokens`, i.e. Vault will consume all the unlocked tokens every time redeemNegativeRewards() called.

This is a part of basker rebalance via `rebalanceBasket -> lockOrUnlockTokens -> unlockTokensFromBasket -> redeemNegativeRewards` sequence:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L276-L280

```solidity
  /// @notice Function to unlock xaver tokens. If tokens are still allocated to protocols they first hevae to be unallocated.
  /// @param _basketId Basket ID (tokenID) in the BasketToken (NFT) contract.
  /// @param _unlockedTokenAmount Amount of derby tokens to unlock and send to the user.
  function unlockTokensFromBasket(uint256 _basketId, uint256 _unlockedTokenAmount) internal {
    uint256 tokensBurned = redeemNegativeRewards(_basketId, _unlockedTokenAmount);
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L406-L418

```solidity
  function lockOrUnlockTokens(uint256 _basketId, int256 _totalDelta) internal {
    if (_totalDelta > 0) {
      lockTokensToBasket(uint256(_totalDelta));
    }
    if (_totalDelta < 0) {
      int256 oldTotal = basketTotalAllocatedTokens(_basketId);
      int256 newTotal = oldTotal + _totalDelta;
      int256 tokensToUnlock = oldTotal - newTotal;
      require(oldTotal >= tokensToUnlock, "Not enough tokens locked");

      unlockTokensFromBasket(_basketId, uint256(tokensToUnlock));
    }
  }
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L318-L330

```solidity
  function rebalanceBasket(
    uint256 _basketId,
    int256[][] memory _deltaAllocations
  ) external onlyBasketOwner(_basketId) nonReentrant {
    uint256 vaultNumber = baskets[_basketId].vaultNumber;
    for (uint k = 0; k < chainIds.length; k++) {
      require(!isXChainRebalancing[vaultNumber][chainIds[k]], "Game: vault is xChainRebalancing");
    }

    addToTotalRewards(_basketId);
    int256 totalDelta = settleDeltaAllocations(_basketId, vaultNumber, _deltaAllocations);

    lockOrUnlockTokens(_basketId, totalDelta);
```

## Tool used

Manual Review

## Recommendation

Consider controlling the new `negativeRewardThreshold` value:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L626-L630

```solidity
  /// @notice Setter for threshold at which user tokens will be sold / burned
  /// @param _threshold treshold in vaultCurrency e.g USDC, must be negative
  function setNegativeRewardThreshold(int256 _threshold) external onlyDao {
+   require(_threshold < 0, "Positive negativeRewardThreshold");
    negativeRewardThreshold = _threshold;
  }
```