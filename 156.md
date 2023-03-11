hyh

medium

# Funds can be frozen on protocol blacklisting

## Summary

When a protocol is blacklisted by Vault's blacklistProtocol() it isn't controlled for full withdrawal of user funds and also there is no rewards claiming performed.

This way both

1) locked funds, i.e. a share of protocol deposit that is generally available, but cannot be withdrawn at the moment due to liquidity situation, and
2) accumulated, but not yet claimed rewards

are frozen within the blacklisted protocol.

## Vulnerability Detail

blacklistProtocol() can be routinely run for a protocol that still has some allocation, but claiming isn't called there and withdrawFromProtocol() performed can end up withdrawing only partial amount of funds (for example because of lending liquidity squeeze).

Protocol blacklisting can be urgent, in which case leaving some funds with the protocol can be somewhat unavoidable, and can be not urgent, in which case locking any meaningful funds with the protocol isn't desirable. Now there is no control lever to distinguish between these scenarios, so the funds can be locked when it can be avoided.

## Impact

Funds attributed to Vault LPs are frozen when a protocol is blacklisted: both reward funds and invested funds that aren't lost, but cannot be withdrawn at the moment due to current liquidity situation of the protocol.

## Code Snippet

blacklistProtocol() performs no rewards claiming and will proceed when withdrawal wasn't full:

https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Vault.sol#L475-L483

```solidity
  /// @notice The DAO should be able to blacklist protocols, the funds should be sent to the vault.
  /// @param _protocolNum Protocol number linked to an underlying vault e.g compound_usdc_01
  function blacklistProtocol(uint256 _protocolNum) external onlyGuardian {
    uint256 balanceProtocol = balanceUnderlying(_protocolNum);
    currentAllocations[_protocolNum] = 0;
    controller.setProtocolBlacklist(vaultNumber, _protocolNum);
    savedTotalUnderlying -= balanceProtocol;
    withdrawFromProtocol(_protocolNum, balanceProtocol);
  }
```

## Tool used

Manual Review

## Recommendation

Consider performing claiming for the blacklisted protocol, for example replace `currentAllocations[_protocolNum] = 0` with logic similar to claimTokens():

```solidity
    if (currentAllocations[_protocolNum] > 0) {
      bool claim = controller.claim(vaultNumber, _protocolNum);
      if (claim) {
        address govToken = controller.getGovToken(vaultNumber, _protocolNum);
        uint256 tokenBalance = IERC20(govToken).balanceOf(address(this));
        Swap.swapTokensMulti(
          Swap.SwapInOut(tokenBalance, govToken, address(vaultCurrency)),
          controller.getUniswapParams(),
          false
        );
      }
      currentAllocations[_protocolNum] = 0;
    }
```

Also, consider introducing an option to revert when funds actually withdrawn from withdrawFromProtocol() are lesser than `balanceProtocol` by more than a threshold, i.e. the flag will block blacklisting if too much funds are being left in the protocol.