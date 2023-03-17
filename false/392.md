HonorLt

medium

# withdrawal request override

## Summary

It is possible that a withdrawal request is overridden during the initial phase.

## Vulnerability Detail

Users have two options to withdraw: directly or request a withdrawal if not enough funds are available at the moment.

When making a `withdrawalRequest` it is required that the user has `withdrawalRequestPeriod` not set:
```solidity
  function withdrawalRequest(
    uint256 _amount
  ) external nonReentrant onlyWhenVaultIsOn returns (uint256 value) {
    UserInfo storage user = userInfo[msg.sender];
    require(user.withdrawalRequestPeriod == 0, "Already a request");

    value = (_amount * exchangeRate) / (10 ** decimals());

    _burn(msg.sender, _amount);

    user.withdrawalAllowance = value;
    user.withdrawalRequestPeriod = rebalancingPeriod;
    totalWithdrawalRequests += value;
  }
```

This will misbehave during the initial period when `rebalancingPeriod` is 0. The check will pass, so if invoked multiple times, it will burn users' shares and overwrite the value.

## Impact

While not very likely to happen, the impact would be huge, because the users who invoke this function several times before the first rebalance, would burn their shares and lose previous `withdrawalAllowance`. The protocol should prevent such mistakes.

## Code Snippet

I have extended one of your test cases to showcase this vulnerability:

```ts
  it('Should not be able to lose previous withdrawal request :(', async function () {
    const { vault, user } = await setupVault();
    await vault.connect(user).deposit(parseUSDC(10_000), user.address); // 10k
    expect(await vault.totalSupply()).to.be.equal(parseUSDC(10_000)); // 10k

    // mocking exchangerate to 0.9
    await vault.setExchangeRateTEST(parseUSDC(0.9));

    // withdrawal request for 2x 5k LP tokens
    await expect(() =>
      vault.connect(user).withdrawalRequest(parseUSDC(5_000)),
    ).to.changeTokenBalance(vault, user, -parseUSDC(5_000));

    await expect(() =>
      vault.connect(user).withdrawalRequest(parseUSDC(5_000)),
    ).to.changeTokenBalance(vault, user, -parseUSDC(5_000));

    // check withdrawalAllowance user and totalsupply
    expect(await vault.connect(user).getWithdrawalAllowance()).to.be.equal(parseUSDC(4_500));
    expect(await vault.totalSupply()).to.be.equal(parseUSDC(0));

    // trying to withdraw allowance before the vault reserved the funds
    await expect(vault.connect(user).withdrawAllowance()).to.be.revertedWith('');

    // mocking vault settings
    await vault.upRebalancingPeriodTEST();
    await vault.setReservedFundsTEST(parseUSDC(10_000));

    // withdraw allowance should give 4.5k USDC
    await expect(() => vault.connect(user).withdrawAllowance()).to.changeTokenBalance(
      IUSDc,
      user,
      parseUSDC(4_500),
    );

    // trying to withdraw allowance again
    await expect(vault.connect(user).withdrawAllowance()).to.be.revertedWith('!Allowance');
  });
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L153

## Tool used

Manual Review

## Recommendation
Require `rebalancingPeriod` != 0 in `withdrawalRequest`, otherwise, force users to directly withdraw.
