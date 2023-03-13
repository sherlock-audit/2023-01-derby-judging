HonorLt

medium

# Not all providers claim the rewards

## Summary

Providers wrongly assume that the protocols will no longer incentivize users with extra rewards.

## Vulnerability Detail

Among the current providers only the `CompoundProvider` claims the `COMP` incentives, others leave the claim function empty:
  ```solidity
    function claim(address _aToken, address _claimer) public override returns (bool) {}
  ```

While many of the protocols currently do not offer extra incentives, it is not safe to assume it will not resume in the future, e.g. when bulls return to the town. For example, `Aave` supports multi rewards claim:
https://docs.aave.com/developers/whats-new/multiple-rewards-and-claim
When it was deployed on Optimism, it offered extra OP rewards for a limited time. There is no guarantee, but a similar thing might happen in the future with new chains and technology.

While `Beta` currently does not have active rewards distribution but based on the schedule, it is likely to resume in the future:
https://betafinance.gitbook.io/betafinance/beta-tokenomics#beta-token-distribution-schedule

## Impact

The implementations of the providers are based on the current situation. They are not flexible enough to support the rewards in case the incentives are back.

## Code Snippet

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/AaveProvider.sol#L115

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Providers/BetaProvider.sol#L122

## Tool used

Manual Review

## Recommendation

Adjust the providers to be ready to claim the rewards if necessary.
