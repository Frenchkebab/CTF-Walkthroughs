---
description: function selector and delegatecall
---

# 6 - Delegation

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Delegate {

  address public owner;

  constructor(address _owner) {
    owner = _owner;
  }

  function pwn() public {
    owner = msg.sender;
  }
}

contract Delegation {

  address public owner;
  Delegate delegate;

  constructor(address _delegateAddress) {
    delegate = Delegate(_delegateAddress);
    owner = msg.sender;
  }

  fallback() external {
    (bool result,) = address(delegate).delegatecall(msg.data);
    if (result) {
      this;
    }
  }
}
```

### Goal of this level

* claim ownership of the contract

### What you should know before

* low-level call\
  \-> see [here](https://www.youtube.com/watch?v=xIAs2S9aCKo)
* function selector\
  \-> see [here](https://www.youtube.com/watch?v=Mn4e4w8h6n8)
* `delegatecall`\
  \-> see [here](https://www.youtube.com/watch?v=uawCDnxFJ-0), [here](https://youtu.be/bqn-HzRclps), and [here](https://youtu.be/bqn-HzRclps)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

Trigger `fallback()` function in `Delegation` contract and run the code of `pwn()`  in the context of `Delegation` contract.

Function selector value of `pwn()` is `"0xdd365b8b"`.

</details>

<pre class="language-javascript"><code class="lang-javascript"><strong>> await contract.sendTransaction({ data: "0xdd365b8b" });
</strong></code></pre>

If you send a transaction to Delegation contract with function selector value of `pwn()` function as data (`"0xdd365b8b"`), \
this will trigger `fallback()` function of **`Delegation`** contract and the message data will be passed to **`Delegate`** contract which eventually triggers `pwn()` function in **Delegate** conract.

Done! 😎

## Key Takeaway

* You can call a function using it's **function selector**.
* If you call contract **`B`**'s function from contract **`A`** using `delegatecall`,\
  code of the function will run in contract A's context.\
  (It will be using storage of contract **`A`**)

