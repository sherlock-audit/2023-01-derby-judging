hyh

high

# Current period profit can be extracted from the Vault by front running state change before exchange rate recalculation

## Summary

Every period `exchangeRate` recalculation can be estimated in advance and this can be used to extract funds from the Vault by depositing right before rebalances with big enough positive effects, earning this period profit without taking the risk.

## Vulnerability Detail

`exchangeRate` can be estimated based on the performance of the underlying pools, all of which are public.

Since `exchangeRate` is recalculated and stored during rebalancing only and old rate is used for deposits until then, it's possible to extract value from the Vault.

Bob the attacker can monitor P&L of the Vault's pools invested and, whenever cumulative realized profit exceeds transaction expenses threshold, do deposit right before rebalancing while `State.Idle` and withdraw the funds right after it. Bob might want to front-run Vault State change or just deposit on observing the accumulated profit.

Even if this be paired with some delays, it is still be low risk / big reward operation for Bob as profit is guaranteed, while risk can be controllable (he can limit the attack in size to be able to withdraw immediately) and aren't high overall (say having earned free 10% on his funds he will need to wait until next rebalancing in order to withdraw; most probably this 10% will cover the associated losses of being invested for one rebalancing period, if any).

POC steps for Bob are:
1) Monitor estimated P&L of rebalancing
2) If big enough Deposit just before rebalancing
3) Withdraw immediately after, if possible, otherwise exit on the next rebalancing

## Impact

It is free profit for the attacker at the expense of depositors, whose already realized profit can be heavily dilluted this way.

## Code Snippet

deposit() before rebalance with current exchangeRate() (which is factually outdated, but used):

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L101-L124

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
```

withdraw() right afterwards rebalancing with new exchangeRate() that is incresed by the realized profit:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L126-L144

```solidity
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

New `exchangeRate` is pushed during rebalancing either by XProvider or Guardian:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L285-L301

```solidity
  /// @notice Step 3 end; xChainController pushes exchangeRate and amount the vaults have to send back to all vaults
  /// @notice Will set the amount to send back to the xController by the xController
  /// @dev Sets the amount and state so the dao can trigger the rebalanceXChain function
  /// @dev When amount == 0 the vault doesnt need to send anything and will wait for funds from the xController
  /// @param _amountToSend amount to send in vaultCurrency
  function setXChainAllocationInt(
    uint256 _amountToSend,
    uint256 _exchangeRate,
    bool _receivingFunds
  ) internal {
    amountToSendXChain = _amountToSend;
    exchangeRate = _exchangeRate;

    if (_amountToSend == 0 && !_receivingFunds) settleReservedFunds();
    else if (_amountToSend == 0 && _receivingFunds) state = State.WaitingForFunds;
    else state = State.SendingFundsXChain;
  }
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L275-L283

```solidity
  /// @notice See setXChainAllocationInt below
  function setXChainAllocation(
    uint256 _amountToSend,
    uint256 _exchangeRate,
    bool _receivingFunds
  ) external onlyXProvider {
    require(state == State.PushedUnderlying, stateError);
    setXChainAllocationInt(_amountToSend, _exchangeRate, _receivingFunds);
  }
```

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L426-L433

```solidity
  /// @notice Step 3: Guardian function
  function setXChainAllocationGuard(
    uint256 _amountToSend,
    uint256 _exchangeRate,
    bool _receivingFunds
  ) external onlyGuardian {
    setXChainAllocationInt(_amountToSend, _exchangeRate, _receivingFunds);
  }
```

## Tool used

Manual Review

## Recommendation

Consider tying the desposits and withdrawals to the next period `exchangeRate`, i.e. gather the requests for both during current period, while processing them all with the updated exchange rate on the next rebalance. Rebalancing period can be somewhat lowered (say to 1 week initialy, then possibly to 3-5 days) to allow for quicker funds turnaround.
