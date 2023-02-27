Gowtham_Ponnana

high

# Reentrancy on mintNewBasket() Function

## Summary
This is where a user can mint his or her basket NFT by calling the following function in the game contract: Where _vaultNumber refers to the vault number for which the user wishes to create a basket.

## Vulnerability Detail: SafeMint() leads to Reentrancy
* Now if you look through the code of _safeMint(), you can see that after the minting, the code makes an external call to the “msg.sender” in order to verify whether the recipient is capable of receiving the tokens or not. 

* If you see here, there is an common exploit of Re-entrancy as the recipient can executed the same function in order to get more tokens than usual. This is an serious issue in nature. 

* This generally happens as the _safeMint is not by default comes with reentrancy guard. 

## Impact: 
 The more Basket NFTs he can mint, the more voting power he gets.

## Code Snippet: 
https://github.com/sherlock-audit/2023-01-derby/blob/main/derby-yield-optimiser/contracts/Game.sol#L255

```solidity
  function mintNewBasket(uint256 _vaultNumber) external returns (uint256) {
    // mint Basket with nrOfUnAllocatedTokens equal to _lockedTokenAmount
    baskets[latestBasketId].vaultNumber = _vaultNumber;
    baskets[latestBasketId].lastRebalancingPeriod = vaults[_vaultNumber].rebalancingPeriod + 1;
    _safeMint(msg.sender, latestBasketId);
    latestBasketId++;

    emit BasketId(msg.sender, latestBasketId - 1);
    return latestBasketId - 1;
  }
```

## Tool used: 
Remix & Ganache

### Manual Review: 
Go through the code so that you can find it somewhat annoying as there is no reentracy guard.

## Recommendations: 

Use Reentrancy guard modifier which can stop these kinds of attacks. 
