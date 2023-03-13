hyh

medium

# Rewards are frozen permanently for any user blacklisted by vault token

## Summary

It is not possible to retrieve the reward funds if staking user (NFT owner) was blacklisted by `vaultCurrency`, for example USDC.

## Vulnerability Detail

If during the duration of a staking the user is blacklisted by the `vaultCurrency`, there is no way to retrieve the accumulated rewards due to that moment.

While MainVault's deposit and withdrawal can take the `recipient` argument, the withdrawRewards() lacks it, so it is not possible to address rewards to anyone else.

As many core tokens are upgradable (USDC, USDT and so forth), such blacklist possibility can be not in place now, but it's possible to introduce it.

## Impact

All reward funds of the user that is still due will be permanently locked with the Vault contract.

## Code Snippet

If Vault currency blacklists LP address and `swapRewards` is off, it's impossible to withdraw, the funds will be frozen within the Vault:

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

Allowance can be created for NFT owner only:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L146-L162

```solidity
  /// @notice Withdrawal request for when the vault doesnt have enough funds available
  /// @dev Will give the user allowance for his funds and pulls the extra funds at the next rebalance
  /// @param _amount Amount to withdraw in LP token
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

## Tool used

Manual Review

## Recommendation

Consider introducing an ability to set the collateral funds `recipient` in withdrawRewards() similarly to deposit() and withdraw():

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L101-L144

```solidity
  /// @notice Deposit in Vault
  /// @dev Deposit VaultCurrency to Vault and mint LP tokens
  /// @param _amount Amount to deposit
  /// @param _receiver Receiving adress for the tokens
  /// @return shares Tokens received by buyer
  function deposit(
    uint256 _amount,
    address _receiver
  ) external nonReentrant onlyWhenVaultIsOn returns (uint256 shares) {
    if (training) {
      require(whitelist[msg.sender]);
      uint256 balanceSender = (balanceOf(msg.sender) * exchangeRate) / (10 ** decimals());
      require(_amount + balanceSender <= maxTrainingDeposit);
    }

    uint256 balanceBefore = getVaultBalance() - reservedFunds;
    vaultCurrency.safeTransferFrom(msg.sender, address(this), _amount);
    uint256 balanceAfter = getVaultBalance() - reservedFunds;

    uint256 amount = balanceAfter - balanceBefore;
    shares = (amount * (10 ** decimals())) / exchangeRate;

    _mint(_receiver, shares);
  }

  /// @notice Withdraw from Vault
  /// @dev Withdraw VaultCurrency from Vault and burn LP tokens
  /// @param _amount Amount to withdraw in LP tokens
  /// @param _receiver Receiving adress for the vaultcurrency
  /// @return value Amount received by seller in vaultCurrency
  function withdraw(
    uint256 _amount,
    address _receiver,
    address _owner
  ) external nonReentrant onlyWhenVaultIsOn returns (uint256 value) {
    value = (_amount * exchangeRate) / (10 ** decimals());

    require(value > 0, "!value");

    require(getVaultBalance() - reservedFunds >= value, "!funds");

    _burn(msg.sender, _amount);
    transferFunds(_receiver, value);
  }
```

