chainNue

high

# inflate initial share price by initial depositor

## Summary

initial deposit can be front-runned by non-whitelist address to inflate share price evading the `training` block, then all users after the first (the attacker) will receive no shares in return for their deposit.

## Vulnerability Detail

`training` block inside `deposit` function intended to be set as true right after deployment. This `training` variable is to make sure the early depositor address is in the whitelist, thus negating any malicious behaviour (especially the first initial depositor) 

```solidity
File: MainVault.sol
106:   function deposit(
107:     uint256 _amount,
108:     address _receiver
109:   ) external nonReentrant onlyWhenVaultIsOn returns (uint256 shares) {
110:     if (training) {
111:       require(whitelist[msg.sender]);
112:       uint256 balanceSender = (balanceOf(msg.sender) * exchangeRate) / (10 ** decimals());
113:       require(_amount + balanceSender <= maxTrainingDeposit);
114:     }
```
First initial depositor issue is pretty well-known issue in vault share-based token minting for initial deposit which is susceptible to manipulation. This issue arise when the initial vault balance is 0, and initial depositor (attacker) can manipulate this share accounting by donating small amount, thus inflate the share price of his deposit. There are a lot of findings about this initial depositor share issue.

Even though the `training` block is (probably) written to mitigate this initial deposit, but since the execution of setting the `training` to be true is not in one transaction, then it's possible to be front-runned by attacker. Then this is again, will make the initial deposit susceptible to attack.

The attack vector and impact is the same as [TOB-YEARN-003](https://github.com/yearn/yearn-security/blob/master/audits/20210719_ToB_yearn_vaultsv2/ToB_-_Yearn_Vault_v_2_Smart_Contracts_Audit_Report.pdf), where users may not receive shares in exchange for their deposits if the total asset amount has been manipulated through a large “donation”.

The initial exchangeRate is a fixed value set on constructor which is not related to totalSupply, but later it will use this totalSupply

```solidity
File: MainVault.sol
64:     exchangeRate = _uScale;
...
290:   function setXChainAllocationInt(
291:     uint256 _amountToSend,
292:     uint256 _exchangeRate,
293:     bool _receivingFunds
294:   ) internal {
295:     amountToSendXChain = _amountToSend;
296:     exchangeRate = _exchangeRate;
297: 
298:     if (_amountToSend == 0 && !_receivingFunds) settleReservedFunds();
299:     else if (_amountToSend == 0 && _receivingFunds) state = State.WaitingForFunds;
300:     else state = State.SendingFundsXChain;
301:   }

File: XChainController.sol
303:     uint256 totalUnderlying = getTotalUnderlyingVault(_vaultNumber) - totalWithdrawalRequests;
304:     uint256 totalSupply = getTotalSupply(_vaultNumber);
305: 
306:     uint256 decimals = xProvider.getDecimals(vault);
307:     uint256 newExchangeRate = (totalUnderlying * (10 ** decimals)) / totalSupply;
```
 
## Impact

initial depositor can inflate share price, other user (next depositor) can lost their asset

## Code Snippet

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L110-L114

```solidity
File: MainVault.sol
106:   function deposit(
107:     uint256 _amount,
108:     address _receiver
109:   ) external nonReentrant onlyWhenVaultIsOn returns (uint256 shares) {
110:     if (training) {
111:       require(whitelist[msg.sender]);
112:       uint256 balanceSender = (balanceOf(msg.sender) * exchangeRate) / (10 ** decimals());
113:       require(_amount + balanceSender <= maxTrainingDeposit);
114:     }
115: 
116:     uint256 balanceBefore = getVaultBalance() - reservedFunds;
117:     vaultCurrency.safeTransferFrom(msg.sender, address(this), _amount);
118:     uint256 balanceAfter = getVaultBalance() - reservedFunds;
119: 
120:     uint256 amount = balanceAfter - balanceBefore;
121:     shares = (amount * (10 ** decimals())) / exchangeRate;
122: 
123:     _mint(_receiver, shares);
124:   }
```
## Tool used

Manual Review

## Recommendation

The simplest way around for this is just set the initial `training` to be `true` either in the variable definition or set it in constructor, so the initial depositor will be from the whitelist.

or, more common solution for this issue is, require a minimum size for the first deposit and burn a portion of the initial shares (or transfer it to a secure address)

