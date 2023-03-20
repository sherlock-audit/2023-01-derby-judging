csanuragjain

medium

# maxTrainingDeposit can be bypassed

## Summary
It was observed that User can bypass the `maxTrainingDeposit` by transferring balance from one user to another

## Vulnerability Detail
1. Observe the `deposit` function

```solidity
function deposit(
    uint256 _amount,
    address _receiver
  ) external nonReentrant onlyWhenVaultIsOn returns (uint256 shares) {
    if (training) {
      require(whitelist[msg.sender]);
      uint256 balanceSender = (balanceOf(msg.sender) * exchangeRate) / (10 ** decimals());
      require(_amount + balanceSender <= maxTrainingDeposit);
    }
...
```

2. So if User balance exceeds maxTrainingDeposit then request fails (considering training is true)
3. Lets say User A has balance of 50 and maxTrainingDeposit is 100
4. If User A deposit amount 51 then it fails since 50+51<=100 is false
5. So User A transfer amount 50 to his another account
6. Now when User A deposit, it does not fail since `0+51<=100`
## Impact
User can bypass maxTrainingDeposit and deposit more than allowed

## Code Snippet
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L113

## Tool used
Manual Review

## Recommendation
If user specific limit is required then transfer should be check below:

```solidity
 require(_amountTransferred + balanceRecepient <= maxTrainingDeposit);
```