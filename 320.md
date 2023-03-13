Jeiwan

medium

# The guardian may not be able to blacklist a protocol

## Summary
An attempt to blacklist a protocol may revert in some situations, not making it possible for the guardian to remove a malicious or a hacked protocol from the system. The protocol will keep receiving funds during rebalancing.
## Vulnerability Detail
The [Vault.blacklistProtocol](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L477) function allows the guardian to blacklist a protocol: a blacklisted protocol [will be excluded from receiving funds during rebalancing](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L188). When a protocol is blacklisted, vault funds need to be [removed from it](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L482) and [updated the cached amount of funds in the protocol](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L481). However, the latter can cause a revert when `balanceUnderlying(_protocolNum)` is greater than `savedTotalUnderlying`, and this can realistically happen in any protocol.

`savedTotalUnderlying` is set at a final step of rebalancing–in the [Vault.rebalance](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L148) function: its value is the sum of all funds deposited to underlying protocols. When a protocol is blacklisted, its underlying balance is [re-calculated and subtracted from `savedTotalUnderlying`](https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L478-L481). In situations when the blacklisted protocol is the only protocol that has allocations (allocations are chosen by gamers based on the performance of different protocols, thus one, the most profitable, protocol can be the only protocol of a vault), it's current underlying balance will be greater than the cached one (`savedTotalUnderlying`) because the protocol will accrue interest between the moment `savedTotalUnderlying` was calculated and the moment it's being blacklisted. Thus, the `balanceProtocol` variable in the `blacklistProtocol` function can be greater than `savedTotalUnderlying`, which will case a revert. The function will keep reverting until there's another protocol with allocations or until there are no allocations in the vault at all (which solely depends on gamers, not the guardian).
## Impact
The guardian may not be able to blacklist a malicious or a hacked protocol, and the protocol will keep receiving funds during rebalancing.
## Code Snippet
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L477-L483
## Tool used
Manual Review
## Recommendation
In the `Vault.blacklistProtocol` function, consider checking if `savedTotalUnderlying` is greater or equal to `balanceProtocol`: if it's not so, consider setting its value to 0, instead of subtracting `balanceProtocol` from it.