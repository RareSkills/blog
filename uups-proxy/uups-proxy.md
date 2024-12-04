# UUPS: Universal Upgradeable Proxy Standard (ERC-1822)

The UUPS pattern is a proxy pattern where the upgrade function resides in the implementation contract, but changes the implementation address stored in the proxy contract via a `delegatecall` from the proxy. The high level mechanism is shown in the animation below:

![Upgrade function of the UUPS Proxy Storage](https://static.wixstatic.com/media/706568_5bf2021cb788446c8901e0e14d85f3b5~mv2.gif)

Similar to the Transparent Upgradeable Proxy, the UUPS pattern solves the problem of function selector clashes by eliminating public functions in the proxy entirely.

This article assumes the reader has read our article on the [Transparent Upgradeable Proxy](https://www.rareskills.io/post/transparent-upgradeable-proxy) pattern.

## The ERC-1967 Proxy Storage Slots Standard

As stated in our article on Transparent Upgradeable Proxy, a functional Ethereum proxy requires at least the following two features:
- A storage slot holding the address of the implementation contract.
- A mechanism for an admin to change the implementation address.

The [ERC-1967 standard](https://www.rareskills.io/post/erc1967) dictates where the storage slot holding the address of the implementation should be held, but does not dictate how to change the address of the implementation, that is, **it leaves the choice of upgrade mechanism up to the developer.**

The UUPS is a proxy pattern in which the mechanism for changing the address of the implementation contract resides in the implementation contract itself rather than in the proxy contract.

This difference is illustrated in the following simplified code:

![Transparent Proxy and UUPS Proxy simplified code differences](https://static.wixstatic.com/media/706568_5cdcdc0a0a5949d9a4fc254de2f8b938~mv2.png/v1/fill/w_740,h_243,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_5cdcdc0a0a5949d9a4fc254de2f8b938~mv2.png)


During an upgrade, the `_upgradeLogic()` function is delegatecalled into the UUPSProxy. Unlike a Transparent Upgradeable Proxy, there is no need for an AdminProxy — a regular EOA can be the admin if desired.

The Transparent Upgradeable Proxy used an AdminProxy to keep the address of the Admin constant. Since a Transparent Upgradeable Proxy has to compare `msg.sender` to the admin on every transaction, it is desirable to compare `msg.sender` to an immutable variable. However, a UUPS proxy only needs to check if `msg.sender` is the admin if they are explicitly calling `_upgradeLogic()` _on the proxy (which delegatecalls_ `_upgradeLogic()` in the implementation).

One of the advantages of this pattern is that the implementation logic itself can be upgraded, that is, the upgradability mechanism can be modified from implementation to implementation. For example, it becomes possible to transition from a simple upgrade logic to a more complex one with voting or timelock mechanisms.

An important tradeoff of this standard is that if an upgrade is made to a new implementation contract that lacks a valid upgrade mechanism, the upgrade chain ends, as it is not possible to move to the next implementation. In other words, since the upgrade mechanism itself may be upgradeable, **there is a risk of breaking the upgrade mechanism.**

To deal with the tradeoff, some proposals have been put forward that first check whether the new implementation contract has a valid upgrade mechanism before migrating to it. The UUPS is one such proposal.

In this article, we will explain how UUPS works in general, examine the OpenZeppelin implementation in detail, and discuss some vulnerabilities that must be taken into consideration when using this pattern.

## UUPS vs Transparent Proxy

OpenZeppelin currently provides implementations for both Transparent and UUPS proxy standards, but [recommends using the later](https://docs.openzeppelin.com/contracts/5.x/api/proxy#transparent-vs-uups). The reason is that, in addition to the flexibility of modifying the upgrade mechanism, the UUPS implementation is lighter and thus consumes less gas during deployment and usage.

This is because there is no need to deploy an administration contract or to check whether the transaction originates from the contract owner, as required in Transparent Proxies. However, each new implementation contract in this pattern does necessitate an upgrade function, which slightly raises the deployment costs of the new implementation contract.

If implementation contracts are running into the 24kb size limit using UUPS, then the Transparent Upgradeable Pattern may be more suitable as it does not need to include the upgrade logic.

## How UUPS works

UUPS was initially defined in [**ERC-1822**](https://eips.ethereum.org/EIPS/eip-1822).

As we saw in the previous section, it is necessary to prevent the proxy contract from accepting a new implementation contract that does not implement the UUPS standard. In other words, any attempt to migrate to a non-UUPS-compliant implementation contract should revert.

### The `proxiableUUID()` function

That is why the standard mandates that every implementation contract incorporates a function with the signature `proxiableUUID()`. **The purpose of this function is to serve as a compatibility check to ensure the new implementation contract adheres to the Universal Upgradeable Proxy Standard.**

The function should return the storage slot where the implementation address is stored. Although the return value is arbitrary and the standard's proponents could have defined the function to return a string like "Hey, I'm UUPS compliant," returning the storage slot is more gas efficient.

The idea is to invoke the `proxiableUUID()` function in the new implementation **before** actually migrating to it. If the new implementation contract has correctly implemented `proxiableUUID()`, it is considered UUPS-compliant, and the migration should continue. Otherwise, the transaction must be rolled back.

The process of both a successful migration and a failed migration attempt are illustrated in the figure below.

![How proxiableUUID function works](https://static.wixstatic.com/media/706568_1cd7b368c8254a86bbb8c257f7e05df1~mv2.png/v1/fill/w_670,h_591,al_c,q_90,enc_auto/706568_1cd7b368c8254a86bbb8c257f7e05df1~mv2.png)

  

### The storage slot

The original proposal suggests the storage slot address to be defined by the formula `keccak256("PROXIABLE")`, however, as the OpenZeppelin implementation uses the ERC-1967 standard, the slot address in their implementation is defined by `keccak256("eip1967.proxy.implementation") - 1`. We'll see this in code shortly.

An animation showing the migration process to a new implementation can be seen below:

<video src="https://video.wixstatic.com/video/706568_950b26d7fab8486fa7987152035037ec/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls>></video>

## Walkthrough of OpenZeppelin UUPS Upgradeable

The contract that implements the UUPS standard in the OpenZeppelin library is named [`UUPSUpgradeable.sol`](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/proxy/utils/UUPSUpgradeable.sol). **This contract should be inherited by the implementation contracts, not by the proxy contract.** The proxy typically inherits from [`ERC1967Proxy`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Proxy.sol), a minimal proxy contract that complies with the ERC-1967 standard.

The purpose of `UUPSUpgradeable.sol` is twofold:
1. It provides the `proxiableUUID()` function, which every implementation must include to be UUPS-compliant,
2. It also provides the `ugradeToAndCall()` function, which is used to migrate to a new implementation. A function with this purpose, as we've seen, must be present in every implementation contract.

### The `proxiableUUID()` function

The `proxiableUUID()` function, which must be called in the new implementation contracts **before** the migrations, is defined as follows, and returns the storage slot of the ERC-1967 standard.

```solidity
function proxiableUUID() external view virtual notDelegated returns (bytes32) {
    return ERC1967Utils.IMPLEMENTATION_SLOT; // conformal to the ERC-1967 standard
}
```

### The `upgradeToAndCall` function

The function responsible for upgrading to the next implementation can have any name. Since it is defined within the implementation contract, there is no risk of function signature collisions, as also occurs in Transparent Proxy.

In `UUPSUpgradeable.sol` this function is named `upgradeToAndCall` and its definition is provided below:

```solidity
function upgradeToAndCall(address newImplementation, bytes memory data) public payable virtual onlyProxy {
    // checks whether the upgrade can proceed
    _authorizeUpgrade(newImplementation); 
    // upgrade to the new implementation
    _upgradeToAndCallUUPS(newImplementation, data); 
}

function _authorizeUpgrade(address newImplementation) internal virtual;

function _upgradeToAndCallUUPS(address newImplementation, bytes memory data) private {
    // checks whether the new implementation implements ERC-1822
    try IERC1822Proxiable(newImplementation).proxiableUUID() returns (bytes32 slot) {
        if (slot != ERC1967Utils.IMPLEMENTATION_SLOT) { 
            revert UUPSUnsupportedProxiableUUID(slot);
        }
        ERC1967Utils.upgradeToAndCall(newImplementation, data);
    } catch {
        // The implementation is not UUPS
        revert ERC1967Utils.ERC1967InvalidImplementation(newImplementation);
    }
}
```

Since it is a public function that should be called only through the active proxy, it has an `onlyProxy` modifier to ensure that.

### The `_authorizeUpgrade` internal function

It is the developer's responsibility to implement the `_authorizeUpgrade` function in the code. This function determines who can perform the upgrade. A straightforward implementation might simply check for ownership to perform the upgrade, as shown below:

```solidity
function _authorizeUpgrade(address newImplementation)
    internal onlyOwner override {}
```

As mentioned earlier, each new implementation can have its own `_authorizeUpgrade` function with unique logic. For instance, if the owner wants to switch to a multi-signature scheme in a new implementation, the necessary code can be included in this function.

Because UUPSUpgradeable is an abstract contract, the code will not compile unless the you explicitly implement `_authorizeUpgrade`.

## Learning about UUPS using Remix

In this section, we will use Remix to gain a clearer visualization of the fundamental workings of UUPS. Although Remix offers advanced deployment features using proxies, we won't use such features here to keep the underlying processes transparent.

Also, to make the code more concise, we will omit initialization functions and modifiers. This will make our code insecure, but the goal is to focus on understanding the core concept of how UUPS operates.

### The proxy contract

Our proxy contract will utilize the [ERC1967Proxy.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Proxy.sol) library, which implements a minimal proxy scheme compliant with the ERC-1967 standard. The address of the initial implementation contract is passed in the constructor. However, the proxy itself lacks a mechanism for updating to a new implementation; this mechanism must be implemented within the implementation contract itself.

<!--![Utilizing the OpenZeppelin ERC1967 in a UUPSProxy contract](https://static.wixstatic.com/media/706568_b7110aaf64354938afdb35fdf60dcfcb~mv2.png/v1/fill/w_740,h_241,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_b7110aaf64354938afdb35fdf60dcfcb~mv2.png)-->

Here is the code to copy and paste:

```solidity
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract UUPSProxy is ERC1967Proxy {

    constructor(address _implementation, bytes memory _data) ERC1967Proxy(_implementation, _data) 
    payable {}
}
```

### The implementation contract

Implementation contracts must inherit from the [**UUPSUpgradeable**](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/proxy/utils/UUPSUpgradeable.sol) contract, which follows the UUPS pattern and includes a mechanism to move to the next implementation. It is essential to override the `_authorizeUpgrade` function, as **the authorization mechanism is not predefined and must be implemented.**

**In the code below, we implement this in a highly insecure manner, as anyone is authorized to perform the upgrade.**

![Insecure Implementation of the UUPS Upgradeable contract (For testing purposes)](https://static.wixstatic.com/media/706568_9fd1d980080541228e04905fb11f99ee~mv2.png/v1/fill/w_740,h_389,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_9fd1d980080541228e04905fb11f99ee~mv2.png)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract UUPSProxy is ERC1967Proxy {
    constructor(address _implementation, bytes memory _data) ERC1967Proxy(_implementation, _data) payable {}
}

// UUPS Implementation

import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract ImplementationOne is UUPSUpgradeable {
    function myNumber() public pure returns (uint256) {
        return 1; // A function to test the implementation
    }

    // In practice, this function should include an onlyOwner modifier
    // or some other form of ownership protection mechanism
    function _authorizeUpgrade(address _newImplementation) internal override {}
}
```

To define an owner for the contract, an initialization function must be created since implementation contracts should not use constructors. You can learn more about this topic in on ur [article on the `Initializable.sol`](https://www.rareskills.io/post/initializable-solidity) contract. Again, here is the code:

```solidity
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract ImplementationOne is UUPSUpgradeable {

    function myNumber() public pure returns (uint256) {
        return 1; // A function to test the implementation
    }
		
    // In practice, this function should include an onlyOwner modifier 
    // or some other form of ownership protection mechanism
    function _authorizeUpgrade(address _newImplementation) internal override {}

}
```

To test the above contracts in Remix, you must follow the following steps:
1. Deploy the implementation contract named `ImplementationOne`.
2. Deploy the proxy contract named `MyProxy`. The constructor requires two parameters: the address of the implementation contract (the address of `ImplementationOne`) and an argument of type `bytes` to initialize the implementation contract. This argument can be `0x`, since it will not be used.
3. To test the contract, open a proxy contract instance using the implementation contract ABI. To do this, in the deployment tab, select the implementation contract, `ImplementationOne`, and enter the proxy contract address in the **At Address** field, as shown in the figure below.

![Deploy at address button in Remix](https://static.wixstatic.com/media/706568_2ac5fdf8f3b141d1b5a1dde9f3e0d1e1~mv2.png/v1/fill/w_451,h_474,al_c,lg_1,q_85,enc_auto/706568_2ac5fdf8f3b141d1b5a1dde9f3e0d1e1~mv2.png)

You will now be able to execute the `myNumber()` function in the implementation contract through the proxy.

### Moving to the next implementation

To move to the next implementation, one must first create a new contract that follows the UUPS pattern, similar to the example below.

<!--![New UUPS Upgradeable Contract example](https://static.wixstatic.com/media/706568_0ce4b48ec90a442287e1ba19b02143cd~mv2.png/v1/fill/w_740,h_587,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_0ce4b48ec90a442287e1ba19b02143cd~mv2.png)-->

```solidity
// UUPS Implementation

import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract ImplementationOne is UUPSUpgradeable {
    function myNumber() public pure returns (uint256) {
        return 1; // A function to test the implementation
    }

    // In practice, this function should include an onlyOwner modifier
    // or some other form of ownership protection mechanism
    function _authorizeUpgrade(address _newImplementation) internal override {}
}

// NEW UUPS Implementation

contract ImplementationTwo is UUPSUpgradeable {

    function myNumber() public pure returns (uint256) {
        return 2; // A function to test the implementation
    }

    // In practice, this function should include an onlyOwner modifier 
    // or some other form of ownership protection mechanism
    function _authorizeUpgrade(address _newImplementation) internal override {}
}
```

After creating the contract, the next steps are as follows:
1. Deploy `ImplementationTwo`.
2. Invoke the `upgradeToAndCall` function on the previous implementation contract through the proxy, passing the address of `ImplementationTwo` as the first parameter (the second parameter can be `0x`). The `proxiableUUID()` function is defined in the parent contract, and its correct return value will be verified before migration.

Attempting to migrate to an implementation that is not UUPS-compliant will fail because the security check mechanism via the `proxiableUUID()` function will prevent it.

## Vulnerabilities in UUPS

### 1. Vulnerability with uninitialized contracts

It is common to initialize implementation contracts. For instance, in the case of an ERC20 upgradeable contract, it is typical to set the token name and symbol during deployment. This procedure is usually done with constructors. However, using constructors in the implementation contract is not helpful, as it would alter the implementation's storage, whereas the 'true' storage resides in the proxy contract.

To initialize implementation contracts, we must rely on regular functions that are configured to execute only once. This can be done using the modifiers provided by OpenZeppelin in the [Initializable.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/utils/Initializable.sol) library.

An example of an initialization function is the code below.

```solidity
function initialize(address initialOwner) initializer public {
    __Ownable_init(initialOwner);
    __UUPSUpgradeable_init();
}
```

### Taking ownership of the implementation

A major vulnerability lies in the fact that this is a public function. It is supposed to be called through the proxy, but it can also be called directly on the implementation contract. Since it sets up the owner of the contract, whoever calls this function first directly on the implementation contract will become the "owner" of that contract.

To clarify, at this point, there will be two owners of the contract:
1. The owner set through the proxy.
2. The "owner" set by calling the implementation contract directly.

![The two owner problem of the UUPS Upgradeable Proxy](https://static.wixstatic.com/media/706568_08866e93e72f45efaa3bfe0c29dd734d~mv2.png/v1/fill/w_738,h_402,al_c,lg_1,q_85,enc_auto/706568_08866e93e72f45efaa3bfe0c29dd734d~mv2.png)

Any function marked as `onlyOwner` will be allowed to be called by either of these two owners. This unintended behavior can pose a risk to the contract, as we will see shortly.

### The solution

The solution to this vulnerability is always to "initialize" the implementation contract by setting the desired state variables directly within the implementation contract. For example, **you should set the owner of the implementation contract or prevent anyone from calling the initialization function directly on the implementation contract.**

OpenZeppelin provides the function `_disableInitializers()` that must be executed in the constructor to achieve this, as shown in the following code:

```solidity
constructor() {
    _disableInitializers();
}
```

### 2. Vulnerability through `delegatecall`

In the implementation contract, you should avoid using `delegatecall` to arbitrary contracts. The biggest risk used to lie in unintentionally making a `delegatecall` to a `selfdestruct` opcode. Since the Cancun fork, `selfdestruct` no longer deletes the contract code. However, the recommendation to avoid using `delegatecall` in implementation contracts remains in place, probably due to use in chains where `selfdestruct` is still active.

A severe vulnerability in OpenZeppelin's UUPS implementation in Contracts v4.1.0 through v4.3.1 was caused by the combination of the two mentioned vulnerabilities:

The code to move to the next implementation, in addition to changing the implementation address, included a `delegatecall` to initialize the new contract. This function, `upgradeToAndCall`, could only be executed by the owner and was intended to be called exclusively through the proxy. However, as mentioned earlier, if the contract wasn't properly "initialized," anyone could assume the role of owner and use the `upgradeToAndCall` function to `delegatecall` to a contract containing a `selfdestruct` opcode.

Due to the vulnerabilities mentioned, OpenZeppelin considers it unsafe to use `delegatecall` within implementation contracts.

## A Checklist for Using UUPS

Here are some guidelines that must be followed when using the UUPS standard in the OpenZeppelin library:
1. Be extremely careful if you override `upgradeToAndCall` to not disrupt the upgrade functionality.
2. Make sure `_authorizeUpgrade` includes an `onlyOwner` modifier or another mechanism that restricts access to authorized accounts only.
3. In an upgrade, be careful when changing the authorization schema in the new version of the implementation contract. For instance, switching to an authorization type where the admin has previously renounced their privileges and this has gone unnoticed.
4. Use the `_disableInitializers()` function in the implementation contract's constructor to prevent initialization.
5. No `delegatecall` or `selfdestruct` is used.

_We would like to thank_ [_ernestognw.eth_](https://x.com/ernestognw) _from_ [_OpenZeppelin_](https://www.openzeppelin.com/) _for reviewing an earlier draft of this work._

*Originally Published August, 26, 2024*
