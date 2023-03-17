Ruhum

high

# Vault executes swaps without slippage protection

## Summary
The vault executes swaps without slippage protection. That will cause a loss of funds because of sandwich attacks.

## Vulnerability Detail
Both in `Vault.claimTokens()` and `MainVault.withdrawRewards()` swaps are executed through the Swaps library. It calculates the slippage parameters itself which doesn't work. Slippage calculations (min out) have to be calculated *outside* of the swap transaction. Otherwise, it uses the already modified pool values to calculate the min out value.

## Impact
Swaps will be sandwiched causing a loss of funds for users you withdraw their rewards.

## Code Snippet
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/libraries/Swap.sol#L60-L97
```sol
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
Slippage parameters should be included in the tx's calldata and passed to the Swap library.