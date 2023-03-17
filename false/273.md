immeas

medium

# `Vault::pullFunds` doesn't pull funds from underlying providers correctly

## Summary
If there isn't enough "free" funds in the vault when needed funds can be pulled from underlying providers. However this might not pull even though there is enough funds.

## Vulnerability Detail
`pullFunds` is intended to pull funds from underlying providers if there either isn't enough liquidity to pay withdrawals or do a rebalance.

It works such that it will pull funds from underlying providers until there is enough funds available in the vault:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L111-L127

Now, imagine that there are two underlying providers with these allocations `[1000,1000e6]`, `minimumPull` is `1e6` and the value needed to pull is `100e6` and for simplicity there's no existing funds in the vault so `vaultCurrency.balanceOf(address(this)) = 0`:

`pullFunds` will check the first provider, since there is an allocation this row passes:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L114

next is `uint256 amountToWithdraw = shortage > balanceProtocol ? balanceProtocol : shortage`

`shortage` is `100e6` and balance protocol `1000` so `amountToWithdraw` will be `1000`.

`1000` will fail on the check `if (amountToWithdraw < minimumPull) break;` and leave the loop and `pullFunds` early even though there were funds available in the second vault.

If the need to pull funds from underlying providers was because of withdrawal request/rewards this might make it impossible for users to withdraw. Since the protocol still believes (rightfully) that there is enough underlying. However it cannot be pulled.

It is also possible for a malicious user to use this to grief as they can allocate very small amounts to "early" providers if they don't have any allocations.

## Impact
Users might not be able to withdraw or funds not balanced properly because there is a low allocation to an underlying provider.

## Code Snippet
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L122

## Tool used
Manual Review

## Recommendation
I recommend these changes:
```diff
diff --git a/derby-yield-optimiser/contracts/Vault.sol b/derby-yield-optimiser/contracts/Vault.sol
index ac9020d..9466d2c 100644
--- a/derby-yield-optimiser/contracts/Vault.sol
+++ b/derby-yield-optimiser/contracts/Vault.sol
@@ -114,12 +114,14 @@ contract Vault is ReentrancyGuard {
       if (currentAllocations[i] == 0) continue;
 
       uint256 shortage = _value - vaultCurrency.balanceOf(address(this));
+      if(shortage < minimumPull) break; // if not enough is missing, leave
+
       uint256 balanceProtocol = balanceUnderlying(i);
 
       uint256 amountToWithdraw = shortage > balanceProtocol ? balanceProtocol : shortage;
       savedTotalUnderlying -= amountToWithdraw;
 
-      if (amountToWithdraw < minimumPull) break;
+      if (amountToWithdraw < minimumPull) continue; // if not enough in this protocol, check next
       withdrawFromProtocol(i, amountToWithdraw);
 
       if (_value <= vaultCurrency.balanceOf(address(this))) break;

```