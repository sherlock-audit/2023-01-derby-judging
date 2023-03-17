tsvetanovv

medium

# Missing functionality to remove whitelist vault

## Summary
[Controller.sol](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Controller.sol#L175) has a whitelist mechanism, but no function to remove the whitelist `vault`.
```solidity
function addVault(address _vault) external onlyDao {
    vaultWhitelist[_vault] = true;
  }
```

## Vulnerability Detail
Once an `vault` address has been added to the whitelist, it is not possible to remove him from the whitelist. 
The vault address may be compromised and need to be removed. It is therefore important to create a mechanism  from which one can remove the whitelist `vault` address.

## Impact
The vault address may be compromised and need to be removed.

## Code Snippet
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Controller.sol#L175

## Tool used

Manual Review

## Recommendation

Add function `removeVault()`:
```solidity
function removeVault(address _vault) external onlyDao {
    vaultWhitelist[_vault] = false;
  }
```