Jeiwan

high

# Rebalancing can be blocked when pulling funds from a TrueFi or a Idle vault

## Summary
An attempt to pull funds from a TrueFi or a Idle vault may revert and cause reverting of the `Vault.rebalance` function. To unlock the function, the team will need to top up the balance of the vault so that there's enough tokens to cover the reserved funds.
## Vulnerability Detail
At the end of a rebalancing, the [Vault.rebalance](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L135) function is called to deposit and/or withdraw funds from protocol according to the allocations set by gamers. The function:
1. [calculates vault's current balance](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L144);
1. [computes the amounts to deposit or withdraw](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L191-L195) and [withdraws excessive amounts from protocols](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L198-L199);
1. [deposits amounts to protocol](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L147);
1. ensures that there's enough tokens to cover the reserved funds and [pulls funds from protocols when it's not enough](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L150).

Notice the order in which the function interacts with the protocols:
1. it withdraws excessive funds;
1. it deposits funds;
1. it withdraws funds when the vault doesn't have enough funds to cover the reserved funds.

However, TrueFi and Idle vaults don't allow depositing and withdrawing (in this order) in the same block. Thus, if `Vault.rebalance` deposits funds to a TrueFi or Idle vault and then withdraws funds to cover reserved funds, there will always be a revert and rebalancing of the vault will be blocked.
## Impact
A rebalancing of a vault can be blocked indefinitely until the vault has enough funds to cover the reserved funds after funds were distributed to protocols.
## Code Snippet
1. `Vault.rebalance` distributes funds to protocols after rebalancing:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L135
1. `executeDeposits` deposits funds to protocols:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L147
1. `pullFunds` withdraws funds from protocols:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L150
1. TrueFi vaults revert when depositing and withdrawing in the same block:
https://github.com/trusttoken/contracts-pre22/blob/76854d53c5036777286d4392495ef28cd5c5173a/contracts/truefi2/TrueFiPool2.sol#L488
```solidity
function join(uint256 amount) external override joiningNotPaused {
    ...
    latestJoinBlock[tx.origin] = block.number;
    ...
}

function liquidExit(uint256 amount) external sync {
    require(block.number != latestJoinBlock[tx.origin], "TrueFiPool: Cannot join and exit in same block");
    ...
}
```
1. Idle vaults revert when depositing and withdrawing in the same block:
https://github.com/Idle-Labs/idle-contracts/blob/develop/contracts/mocks/IdleTokenV3_1NoConst.sol#L630
```solidity
function mintIdleToken(uint256 _amount, bool, address _referral)
  external nonReentrant whenNotPaused
  returns (uint256 mintedTokens) {
  _minterBlock = keccak256(abi.encodePacked(tx.origin, block.number));
  ...
  }

function _redeemIdleToken(uint256 _amount, bool[] memory _skipGovTokenRedeem)
  internal nonReentrant
  returns (uint256 redeemedTokens) {
    _checkMintRedeemSameTx();
    ...
  }

function _checkMintRedeemSameTx() internal view {
  require(keccak256(abi.encodePacked(tx.origin, block.number)) != _minterBlock, "REE");
}
```
## Tool used
Manual Review
## Recommendation
Consider improving the logic of the `Vault.rebalance` function so that funds are never withdrawn from protocols after they have been deposited to protocols.