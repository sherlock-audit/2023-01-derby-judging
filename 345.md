bin2chen

medium

# withdrawAllowance() reservedFunds there will be residue

## Summary
 reservedFunds there will be residue, maybe accumulate multiple error values and no one may withdraw it
## Vulnerability Detail
When the user executes withdrawAllowance (), if balance is not enough, the withdrawal error is less than maxDivergenceWithdraws, the error can be ignored
The code is as follows:
```solidity
  function withdrawAllowance() external nonReentrant onlyWhenIdle returns (uint256 value) {
    UserInfo storage user = userInfo[msg.sender];
    require(user.withdrawalAllowance > 0, allowanceError);
    require(rebalancingPeriod > user.withdrawalRequestPeriod, "Funds not arrived");

    value = user.withdrawalAllowance;
    value = checkForBalance(value);

    reservedFunds -= value;  //<------subtract value
    delete user.withdrawalAllowance;  //<------but withdrawalAllowance will be clear
    delete user.withdrawalRequestPeriod;

    transferFunds(msg.sender, value);
  }

```
There is a problem here, `reservedFunds` subtracting the value of after the error  ` reservedFunds -= value`
but the actual clear is before the error ` delete user.withdrawalAllowance`
`reservedFunds` may always have residuals that no one can withdrawal
Example：
Assumptions
1.alice withdraw request userInfo[alice].withdrawalAllowance = 100
2.reservedFunds = totalWithdrawalRequests = 100
3.alice call withdrawAllowance() ,but balance only 99 (maxDivergenceWithdraws is ok)
so alice gets 99
userInfo[alice].withdrawalAllowance=0
reservedFunds = 1

4. other period, same one have totalWithdrawalRequests=1000
so reservedFunds = 1001

If the error problem occurs several times
`reservedFunds` will accumulate multiple error values and no one can withdraw it
Suggest `reservedFunds` subtracting `user.withdrawalAllowance`  substitute for `value`

## Impact

`reservedFunds` maybe accumulate multiple error values and no one can withdraw it

## Code Snippet

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L166-L179

## Tool used

Manual Review

## Recommendation
```solidity
  function withdrawAllowance() external nonReentrant onlyWhenIdle returns (uint256 value) {
    UserInfo storage user = userInfo[msg.sender];
    require(user.withdrawalAllowance > 0, allowanceError);
    require(rebalancingPeriod > user.withdrawalRequestPeriod, "Funds not arrived");

    value = user.withdrawalAllowance;
    value = checkForBalance(value);

-   reservedFunds -= value;
+   reservedFunds -= user.withdrawalAllowance; 
    delete user.withdrawalAllowance;
    delete user.withdrawalRequestPeriod;

    transferFunds(msg.sender, value);
  }
```
