---
description: calculating contract address
---

# 17 - Recovery

Ethernaut Level17: [Recovery](https://ethernaut.openzeppelin.com/level/0xb4B157C7c4b0921065Dded675dFe10759EecaA6D)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Recovery {

  //generate tokens
  function generateToken(string memory _name, uint256 _initialSupply) public {
    new SimpleToken(_name, msg.sender, _initialSupply);
  
  }
}

contract SimpleToken {

  string public name;
  mapping (address => uint) public balances;

  // constructor
  constructor(string memory _name, address _creator, uint256 _initialSupply) {
    name = _name;
    balances[_creator] = _initialSupply;
  }

  // collect ether in return for tokens
  receive() external payable {
    balances[msg.sender] = msg.value * 10;
  }

  // allow transfers of tokens
  function transfer(address _to, uint _amount) public { 
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] = balances[msg.sender] - _amount;
    balances[_to] = _amount;
  }

  // clean up after ourselves
  function destroy(address payable _to) public {
    selfdestruct(_to);
  }
}
```

### Goal of this level

* recover (or remove) the `0.001` ether from the lost contract address

### What you should know before

* How the address of an Ethereum contract is computed\
  \-> see [here](https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed)&#x20;
* RLP Encoding\
  \-> see [here](https://medium.com/coinmonks/data-structure-in-ethereum-episode-1-recursive-length-prefix-rlp-encoding-decoding-d1016832f919)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

We will recalculate the address of the contract using instance address and its nonce.

</details>

There are 2 ways to solve this level.

1. Recalculate the address from instance address and its nonce
2. Search instance addresss on **Etherscan**&#x20;

Using Etherscan might be way easier, but we will solve this level by recalculating the address.

```javascript
> const recoveredAddr = _ethers.utils.getContractAddress({
  from: instance,
  nonce: '1',
});
```

`_ethers.utils.getContractAddress` gets address, nonce as input and calculates `rightmost_20_bytes(keccak(RLP(sender address, nonce)))` under the hood.

Now we know the lost address.

We just have to call `destroy()` function.

```javascript
> const functionSelector = web3.eth.abi.encodeFunctionSignature("destroy(address)");

> const param = web3.eth.abi.encodeParameter("address", player).substr(2);

> const calldata = functionSelector + param;

> await web3.eth.sendTransaction({
    from: player,
    to: recoveredAddr,
    data: calldata
});
```

Done! 😎

## Key Takeaways

* contract address can be calculated with deployer address and its nonce.
