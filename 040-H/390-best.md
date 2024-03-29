c7e7eff

high

# Anyone can execute certain functions that use cross chain messages and potentially cancel them with potential loss of funds.

c7e7eff
High

## Summary
Certain functions that route messages cross chain on the `Game` and `MainVault` contract are unprotected (anyone can call them under the required state of the vaults). The way the cross chain messaging is implemented in the XProvider makes use of Connext's `xcall()` and sets the `msg.sender` as the `delegate` and `msg.value` as `relayerFee`.
There are two possible attack vectors with this:
- Either an attacker can call the function and set the msg.value to low so it won't be relayed until someone bumps the fee (Connext allows anyone to bump the fee). This however means special action must be taken to bump the fee in such a case.
- Or the attacker can call the function  (which irreversibly changes the state of the contract) and as the delegate of the `xcall` cancel the message. This functionality is however not yet active on Connext, but the moment it is the attacker will be able to change the state of the contract on the origin chain and make the cross chain message not execute on the destination chain leaving the contracts on the two chains out of synch with possible loss of funds as a result.

## Vulnerability Detail
The `XProvider` contract's `xsend()`  function sets the `msg.sender` as the delegate and `msg.value` as `relayerFee`
```solidity
    uint256 relayerFee = _relayerFee != 0 ? _relayerFee : msg.value;
    IConnext(connext).xcall{value: relayerFee}(
      _destinationDomain, // _destination: Domain ID of the destination chain
      target, // _to: address of the target contract
      address(0), // _asset: use address zero for 0-value transfers
      msg.sender, // _delegate: address that can revert or forceLocal on destination
      0, // _amount: 0 because no funds are being transferred
      0, // _slippage: can be anything between 0-10000 because no funds are being transferred
      _callData // _callData: the encoded calldata to send
    );
  }
```

`xTransfer()` using `msg.sender` as delegate:
```solidity
    IConnext(connext).xcall{value: (msg.value - _relayerFee)}(
      _destinationDomain, // _destination: Domain ID of the destination chain
      _recipient, // _to: address receiving the funds on the destination
      _token, // _asset: address of the token contract
      msg.sender, // _delegate: address that can revert or forceLocal on destination
      _amount, // _amount: amount of tokens to transfer
      _slippage, // _slippage: the maximum amount of slippage the user will accept in BPS (e.g. 30 = 0.3%)
      bytes("") // _callData: empty bytes because we're only sending funds
    );
  }
```

Connext [documentation](https://docs.connext.network/developers/reference/SDK/sdk-base#parameters-9) explaining:
```solidity
params.delegate | (optional) Address allowed to cancel an xcall on destination.
```
Connext [documentation](https://docs.connext.network/developers/guides/handling-failures#high-slippage) seems to indicate this functionality isn't active yet though it isn't clear whether that applies to the cancel  itself or only the bridging back the funds to the origin chain.

## Impact
An attacker can call certain functions which leave the relying contracts on different chains in an unsynched state, with possible loss of funds as a result (mainly on `XChainControleler`'s `sendFundsToVault()` when actual funds are transferred.

## Code Snippet
`MainVault`'s `pushTotalUnderlyingToController() `has no access control and calls `pushTotalUnderlying()` on the `xProvider`
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L249

`xProvider`'s `pushTotalUnderlying()` calling `xsend()`
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L243

`MainVault`'s `sendRewardsToGame()` has no access control and calls `pushRewardsToGame()` on the `xProvider`
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/MainVault.sol#L365

`xProvider`'s `pushRewardsToGame()` calling `xsend()`
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L443

`XProvider` setting `msg.sender` as `delegate` and `msg.value` as relayer fee:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L115-L124

`XChainController`'s unprotected `sendFundsToVault()`
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XChainController.sol#L409-L414

`XProvider`'s `xTransfer()` setting `msg.sender` as `delegate`:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/XProvider.sol#L152-L161

Also on the  `Game` contract, the unprotected `psuhAllocationsToController()` function:
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L424

and the `pushAllocationsToVaults()`
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L465

## Tool used
Manual Review

## Recommendation
Provide access control limits to the functions sending message across Connext so only the Guardian can call these functions with the correct msg.value and do not use msg.sender as a delegate but rather a configurable address like the Guardian.

