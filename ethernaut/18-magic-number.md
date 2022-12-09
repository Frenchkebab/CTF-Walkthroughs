---
description: understanding bytecode and opcodes
---

# 18 - Magic Number

Ethernaut Level18: [Magic Number](https://ethernaut.openzeppelin.com/level/0xFe18db6501719Ab506683656AAf2F80243F8D0c0)

{% hint style="info" %}
Recommend you to solve EVM Puzzles before solving this level.\
It help you understand better about how opcodes works.
{% endhint %}

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract MagicNum {

  address public solver;

  constructor() {}

  function setSolver(address _solver) public {
    solver = _solver;
  }

  /*
    ____________/\\\_______/\\\\\\\\\_____        
     __________/\\\\\_____/\\\///////\\\___       
      ________/\\\/\\\____\///______\//\\\__      
       ______/\\\/\/\\\______________/\\\/___     
        ____/\\\/__\/\\\___________/\\\//_____    
         __/\\\\\\\\\\\\\\\\_____/\\\//________   
          _\///////////\\\//____/\\\/___________  
           ___________\/\\\_____/\\\\\\\\\\\\\\\_ 
            ___________\///_____\///////////////__
  */
}
```

### Goal of this level

* deploy a contract that returns 42 that uses 10 opcodes at most

### What you should know before

* opcode \
  \-> see [here](https://www.evm.codes/?fork=merge)
* how contract is deployed in terms of bytecode\
  \-> see [here](https://blog.openzeppelin.com/deconstructing-a-solidity-contract-part-i-introduction-832efd2d7737/)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

We will deploy a contract that returns 42

</details>

Before reading this solution, I highly recommend you to read [this](https://blog.openzeppelin.com/deconstructing-a-solidity-contract-part-i-introduction-832efd2d7737/) article.

```
600a600c600039600a6000f3602A60805260206080f3
```

This is the code of the contract we will deploy.

Let's see what it does.

```
[00]	PUSH1	0a
[02]	PUSH1	0c
[04]	PUSH1	00
[06]	CODECOPY	
[07]	PUSH1	0a
[09]	PUSH1	00
[0b]	RETURN	
[0c]	PUSH1	2A
[0e]	PUSH1	80
[10]	MSTORE	
[11]	PUSH1	20
[13]	PUSH1	80
[15]	RETURN	

```



```
[0c]	PUSH1	2A
[0e]	PUSH1	80
[10]	MSTORE	
[11]	PUSH1	20
[13]	PUSH1	80
[15]	RETURN	
```

This is the runtime code of our contract.

Whenever the contract is called, it will store value `0x2A` (`42` in dec) on the memory at `0x80` .

And it returns `0x20` bytes from `0x80`. \
(which is what we just stored)

```
[00]	PUSH1	0a
[02]	PUSH1	0c
[04]	PUSH1	00  
[06]	CODECOPY	
[07]	PUSH1	0a
[09]	PUSH1	00
[0b]	RETURN	
```

This part, the cocde is copying `0x0a` bytes (which is the length of the runtime code above) from `0x0c` (where the runtime code starts) on the memory location `0x00`.

Then it is returning that `0x0A` bytes from memory loaction `0x00`.

You can see more details for each opcode here: [https://www.evm.codes/](https://www.evm.codes/)&#x20;



```javascript
> const bytecode = "0x600a600c600039600a6000f3602A60805260206080f3";

> await web3.eth.sendTransaction({
    from: player,
    data: bytecode,
    (err, res) => { console.log(res) }
});

// address of deployed bytecode contract
> await contract.setSolver("0xcDa438124cB513E26439abb73576F1d9C05e58dB");
```

Done! 😎
