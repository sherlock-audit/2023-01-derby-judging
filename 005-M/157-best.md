hyh

medium

# Vault rewards withdrawal swapping is subject to sandwich attack

## Summary

withdrawRewards() calls Swap.swapTokensMulti(), that utilizes `quoteExactInput` for minimum accepted output estimation to be used with `router.exactInput` calls. This setup allows for sandwich attacks, where pool is being skewed before such swap call and returned back thereafter.

## Vulnerability Detail

This is possible as both quoting and swapping depend on the state of the pool and can be manipulated by attacker, i.e. such approach performs slippage control on an already skewed pool.

As a result, swaps withdrawRewards() perform when `swapRewards` is enabled can happen at manipulated price and withdrawing user end up receiving fewer Derby tokens than current market price dictates.

## Impact

Net impact is a partial fund loss whenever `user.rewardAllowance` is big enough as sandwich attacks are common and can be counted to happen almost always as long as economic viability is here, i.e. when the attacker's profit, which is linear to the amount being swapped, is big enough to cover swap costs.

## Code Snippet

Caller cannot control the result of swapTokensMulti(), which is performed on every withdrawal when `swapRewards` is set:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L206-L230

```solidity
  /// @notice Withdraw the reward allowance set by the game with redeemRewardsGame
  /// @dev Will swap vaultCurrency to Derby tokens, send the user funds and reset the allowance
  function withdrawRewards() external nonReentrant onlyWhenIdle returns (uint256 value) {
    UserInfo storage user = userInfo[msg.sender];
    require(user.rewardAllowance > 0, allowanceError);
    require(rebalancingPeriod > user.rewardRequestPeriod, "!Funds");

    value = user.rewardAllowance;
    value = checkForBalance(value);

    reservedFunds -= value;
    delete user.rewardAllowance;
    delete user.rewardRequestPeriod;

    if (swapRewards) {
      uint256 tokensReceived = Swap.swapTokensMulti(
        Swap.SwapInOut(value, address(vaultCurrency), derbyToken),
        controller.getUniswapParams(),
        true
      );
      IERC20(derbyToken).safeTransfer(msg.sender, tokensReceived);
    } else {
      vaultCurrency.safeTransfer(msg.sender, value);
    }
  }
```

swapTokensMulti() set `amountOutMinimum` to result of quoteExactInput(), which depends on the state of the pool and can be manipulated:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/libraries/Swap.sol#L60-L97

```solidity
  /// @notice Swap tokens on Uniswap
  /// @param _swap Number of tokens to sell, token to sell, token to receive
  /// @param _uniswap Address of uniswapRouter, uniswapQuoter and poolfee
  /// @return Amountout Number of tokens received
  function swapTokensMulti(
    SwapInOut memory _swap,
    IController.UniswapParams memory _uniswap,
    bool _rewardSwap
  ) public returns (uint256) {
    IERC20(_swap.tokenIn).safeIncreaseAllowance(_uniswap.router, _swap.amount);

    uint256 amountOutMinimum = IQuoter(_uniswap.quoter).quoteExactInput(
      abi.encodePacked(_swap.tokenIn, _uniswap.poolFee, WETH, _uniswap.poolFee, _swap.tokenOut),
      _swap.amount
    );

    uint256 balanceBefore = IERC20(_swap.tokenOut).balanceOf(address(this));
    if (_rewardSwap && balanceBefore > amountOutMinimum) return amountOutMinimum;

    ISwapRouter.ExactInputParams memory params = ISwapRouter.ExactInputParams({
      path: abi.encodePacked(
        _swap.tokenIn,
        _uniswap.poolFee,
        WETH,
        _uniswap.poolFee,
        _swap.tokenOut
      ),
      recipient: address(this),
      deadline: block.timestamp,
      amountIn: _swap.amount,
      amountOutMinimum: amountOutMinimum
    });

    ISwapRouter(_uniswap.router).exactInput(params);
    uint256 balanceAfter = IERC20(_swap.tokenOut).balanceOf(address(this));

    return balanceAfter - balanceBefore;
  }
```

## Tool used

Manual Review

## Recommendation

Consider introducing the minimum accepted return argument to the withdrawRewards() that is then passed to swapTokensMulti(), where then end amount of Derby tokens received is controlled with it so user can limit the realized slippage.