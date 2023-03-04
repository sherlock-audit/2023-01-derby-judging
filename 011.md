Bobface

high

# Underlying protocol additional tokens can be stolen by sandwich attack

## Summary
[`Vault.claimTokens`](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L404) collects additional tokens paid out by the underlying protocols and swaps them to the vault token. The contract does not check for the pool to be manipulated, allowing for the swap to be sandwiched and thus the tokens to be stolen.

*Note that I submitted a similar issue regarding stealing user rewards. This report regards a similar bug, yet I believe it to be a separate bug, since the attack path is different, and fixing the other issue might not fix this one.* 

## Vulnerability Detail
When the vault accumulates additional tokens from the underlying protocols, [`Vault.claimTokens`](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L404) swaps these tokens to the vault token using Uniswap V3.

The function first calculates the expected output amount from the current on-chain pool state:
```solidity
// libraries/Swap.sol @ function swapTokensMulti
uint256 amountOutMinimum = IQuoter(_uniswap.quoter).quoteExactInput(
     abi.encodePacked(_swap.tokenIn, _uniswap.poolFee, WETH, _uniswap.poolFee, _swap.tokenOut),
    _swap.amount
);
```

and then swaps the tokens:
```solidity
// libraries/Swap.sol @ function swapTokensMulti
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
```

The resulting token amount is accepted without further check:
```solidity
// libraries/Swap.sol @ function swapTokensMulti
return balanceAfter - balanceBefore;
```

The issue is that there is no check for whether the pool is in a manipulated state. The code simply calculates the amount it will get as output from the **current pool state** and accepts that as minimum. The pool can however already be heavily manipulated.

## Impact
This issue allows for the well-known sandwich attack:
1. A MEV bot or attacker sees a call to `claimTokens` pending in the mempool, or he submits it himself since the function is open to be called by anyone.
2. He frontruns this transaction with his own, where he dumps tokens into the respective pools.
3. The `claimTokens` transaction gets executed, which does not check for the pool to be manipulated, and thus further sells tokens into the pools, receiving nearly no tokens in return since the pools are already heavily dumped.
4. The attacker backruns the `claimTokens` transaction and uses the funds he received from step 2 to buy back funds from the pool. He will receive more than he initially spent, and the profit he makes will be the amount of tokens the vault initially earned.
## Tool used

Manual Review

## Recommendation
Implement protection against manipulated pools. There are various ways to solve this issue, the most common ones include:
- use an off-chain oracle such as Chainlink to check whether the current pool price deviates too much from the expected price
- use an on-chain oracle such as Uniswap's TWAP oracle to check whether the current pool price deviates too much from the expected price
- let the caller pass a `minAmountOut` parameter to the function, which checks that the final output in tokens is not too low. Note that since this function is callable by anyone, this would also require the access to the function to be restricted.


## Code Snippet
PoC omitted in this case since it should be clearly recognizable that there is no such protection in place and sandwich attacks are a well-known concept. 