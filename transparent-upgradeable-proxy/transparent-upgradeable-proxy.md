# The Transparent Upgradeable Proxy Pattern Explained in Detail

The Transparent Upgradeable Proxy is a design pattern for upgrading a proxy while eliminating the possibility of a function selector clash.

A functional Ethereum proxy needs at least the following two features:

- A storage slot holding the address of the implementation contract
- A mechanism for an admin to change the implementation address

The [ERC-1967](https://www.rareskills.io/post/erc1967) standard dictates where the storage slot holding the address of the implementation should be held so that the chance of a storage collision is minimized. However, the ERC-1967 standard does not dictate how to change the address of the implementation.

The problem with putting an additional function inside the proxy to change the implementation (such as `updateImplementation(address _newImplementation)`) is that the update function has a non-negligible chance of clashing with a function in the implementation.

## Function Selector Clashing

Declaring public functions inside the proxy to update the implementation address introduces the possibility of a [function selector](https://www.rareskills.io/post/function-selector) clash.

Here is a trivial example:

```solidity
contract ProxyUnsafe {

    function changeImplementation(
            address newImplementation
    ) public {
        // some code...
    }

    fallback(bytes calldata data) external payable (bytes memory) {
        (bool ok, bytes memory data) = getImplementation().delegatecall(data);
        require(ok, "delegatecall failed");
        return data;
    }
}

contract Implementation {
    // an identical function is declared here -- they will clash
    function changeImplementation(
            address newImplementation
    ) public {

    }
    //...
}
```

**Remember, the fallback is always checked last.** Before the fallback is called, the Proxy contract will check if the 4 byte function selector matches `changeImplementation` (or any other public function in the proxy).

Therefore, if a public function is declared in the proxy, there are two kinds of function selector clashes that can occur:

1. If the implementation contract implements a function with the same signature, that function will be uncallable because the proxy's public function with the same signature will be called, not the fallback. And if the fallback is not triggered, there won't be a [delegatecall](https://www.rareskills.io/post/delegatecall) to the implementation.
2. If the implementation contract has a function with the same function selector as the public function in the proxy, it will also be uncallable for the same reason. This scenario is a function selector clash that can happen by random chance when the four-bytes match. The probability of the two different functions having the same selector is 1 in 4.29 billion; a function selector consists of 4 bytes, so there are 4.29 billion possibilities. That is a small probability, but *not* negligible. For example, `clash550254402()` has the same function selector as `proxyAdmin()`.

## The Transparent Upgradeable Proxy Pattern Prevents Function Selector Clashing Entirely

The Transparent Upgradeable Proxy Pattern is a design pattern to completely eliminate the possibility of function selector clashing.

Specifically, the Transparent Upgradeable Proxy Pattern dictates **that there should be no public functions on the proxy except the fallback**.

But with only a fallback function, how do we call the function to upgrade the proxy?

The answer is to detect if `msg.sender` is the admin.

```solidity
contract Proxy is ERC1967 {
    address immutable admin;

    constructor(address admin_) {
        admin = admin_
    }

    fallback() external payable {
        if (msg.sender == admin) {
            // upgrade logic
        } else {
            // delegatecall to implementation
        }
    }
}
```

The implication of this is that the admin cannot directly use the proxy since their calls are always routed away from the delegatecall. However, using a different mechanism we will discuss later, it is still possible for the admin to call the proxy and the proxy to delegatecall to the implementation as a normal transaction.

## Changing an immutable admin

In the code snippet above, the admin is immutable. This means that the contract technically does not comply with ERC-1967 which states that the admin must be kept in storage slot `0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103` or `bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1)`.

To be compatible with ERC-1967, the Transparent Upgradeable Proxy stores the address of the admin in that storage slot, *but it does not use that storage variable*.

The presence of an address in that storage slot will signal to block explorers that the contract is a proxy (one of the intents of ERC-1967). However, reading from storage every time a call to the proxy is made adds another 2,100 gas to the call. Hence, it is desirable to use an immutable variable.

## "Changing" the admin

However, it is still desirable to be able to update the admin address — but initially this seems impossible because the Proxy uses an immutable variable.

The approach of the Transparent Upgradeable Proxy to enable the alteration of the proxy contract admin is twofold. First, it designates another contract, known as `ProxyAdmin`, to be the admin of the proxy contract.

![Diagram showing the relationship of proxy, proxyadmin and owner in transparent upgradeable proxy](https://static.wixstatic.com/media/706568_69c82d0d1cbc4e74a6bf7026d28b58c1~mv2.jpg/v1/fill/w_740,h_738,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_69c82d0d1cbc4e74a6bf7026d28b58c1~mv2.jpg)

A smart contract's address will never change, so this is compatible with the Transparent Upgradeable Proxy storing the address of the admin in an immutable variable.

Second, the *owner* of `ProxyAdmin` is the "true" admin. The `ProxyAdmin` simply routes calls from the `owner` to the `Proxy`. The "true" admin calls the `ProxyAdmin` and the `ProxyAdmin` calls the Transparent Proxy. By changing the owner of the `ProxyAdmin` we can change who has the ability to upgrade the Transparent Proxy.

## AdminProxy

Below is the code from the [OpenZeppelin AdminProxy](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/transparent/ProxyAdmin.sol) (with the comments removed). Observe that there is only a single function `upgradeAndCall()` which can only call `upgradeToAndCall()` on the Proxy.

```solidity
pragma solidity ^0.8.20;

import {ITransparentUpgradeableProxy} from "./TransparentUpgradeableProxy.sol";
import {Ownable} from "../../access/Ownable.sol";

contract ProxyAdmin is Ownable {
    string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";

    constructor(address initialOwner) Ownable(initialOwner) {}

    function upgradeAndCall(
        ITransparentUpgradeableProxy proxy,
        address implementation,
        bytes memory data
    ) public payable virtual onlyOwner {
        proxy.upgradeToAndCall{value: msg.value}(implementation, data);
    }
}
```

There is a common misconception that the admin of the Transparent Proxy is not able to use the contract because their calls get forwarded to the upgrade. However, the `owner` of the `AdminProxy` can use the `Proxy` with no issues, as the diagram below demonstrates.

In fact, we will see later there is a mechanism for the `ProxyAdmin` to make an arbitrary call to the proxy, as the function name `upgradeToAndCall()` implies.

![Diagram of possible call routes for owner and user in a transparent upgradeable proxy](https://static.wixstatic.com/media/706568_13cb902dce8741acb57688dbe9f5ce40~mv2.jpg/v1/fill/w_740,h_370,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/706568_13cb902dce8741acb57688dbe9f5ce40~mv2.jpg)

### Making the proxy non-upgradeable

If the `owner` is changed to be address zero, or another smart contract that is not able to properly use the function `upgradeAndCall()` (or change the `owner`), then the Transparent Upgradeable Proxy won't be upgradeable anymore. This can happen, for example, if the owner of the AdminProxy is set to be a different AdminProxy contract.

## Implementation Details

The OpenZeppelin Transparent Upgradeable Proxy implements the standard with three contracts:

- Proxy.sol
- ERC1967Proxy.sol (inherits Proxy.sol)
- TransparentUpgradeableProxy.sol (inherits ERC1967Proxy.sol)

### Parentmost contract: Proxy.sol

The base contract is [Proxy.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/Proxy.sol). Given an implementation address, it sends a delegatecall to the implementation. The `_implementation()` function is not implemented in `Proxy` — it is overridden and implemented by its child `ERC1967Proxy` which will make it return the relevant storage slot.

```solidity
abstract contract Proxy {
    function _delegate(address implementation) internal virtual {
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), implementation, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())

            switch result
            case 0 {
                revert(0, returndatasize())
            }
            default {
                return(0, returndatasize())
            }
        }
    }

    function _implementation() internal view virtual returns (address);

    function _fallback() internal virtual {
        _delegate(_implementation());
    }

    fallback() external payable virtual {
        _fallback();
    }
}
```

### Child of Proxy.sol: ERC1967Proxy.sol

[ERC1967Proxy.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Proxy.sol) inherits from `Proxy.sol`. This adds (and overrides) the internal `_implementation()` function which returns the implementation address stored at the slot specified by ERC-1967. The constructor of this contract stores the implementation in the storage slot specified ERC-1967. However, the Transparent Upgradeable Proxy will not make use of this function — instead using its own immutable variable.

```solidity
pragma solidity ^0.8.20;

import {Proxy} from "../Proxy.sol";
import {ERC1967Utils} from "./ERC1967Utils.sol";

contract ERC1967Proxy is Proxy {

    constructor(address implementation, bytes memory _data) payable {
        ERC1967Utils.upgradeToAndCall(implementation, _data);
    }

    // reads from bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)
    function _implementation() internal view virtual override returns (address) {
        return ERC1967Utils.getImplementation();
    }
}
```

### Child of ERC1967Proxy.sol: TransparentUpgradeableProxy.sol

Finally, [TransparentUpgradeableProxy.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/transparent/TransparentUpgradeableProxy.sol) inherits from ERC1967Proxy.sol. In the constructor of this contract, the `ProxyAdmin` is deployed and the immutable admin (first variable in the contract) is set to be the address of the `ProxyAdmin` in the constructor.

```solidity
contract TransparentUpgradeableProxy is ERC1967Proxy {
    address private immutable _admin;

    error ProxyDeniedAdminAccess();

    constructor(address _logic, address initialOwner, bytes memory _data) payable ERC1967Proxy(_logic, _data) {
        _admin = address(new ProxyAdmin(initialOwner));
        // Set the storage value and emit an event for ERC-1967 compatibility
        ERC1967Utils.changeAdmin(_proxyAdmin());
    }

    function _proxyAdmin() internal view virtual returns (address) {
        return _admin;
    }

    function _fallback() internal virtual override {
        if (msg.sender == _proxyAdmin()) {
            if (msg.sig != ITransparentUpgradeableProxy.upgradeToAndCall.selector) {
                revert ProxyDeniedAdminAccess();
            } else {
                _dispatchUpgradeToAndCall();
            }
        } else {
            super._fallback();
        }
    }

    function _dispatchUpgradeToAndCall() private {
        (address newImplementation, bytes memory data) = abi.decode(msg.data[4:], (address, bytes));
        ERC1967Utils.upgradeToAndCall(newImplementation, data);
    }
}
```

Let's consider the case where `msg.sender` is the `_proxyAdmin`. In that case, the call gets routed to `_dispatchUpgradeToAndCall()`, but `_fallback()` first checks that the function selector provided is the function selector for `upgradeToAndCall`. The "selector" here is not a "real" selector, since the Transparent Upgradeable Proxy does not have public functions. However, to allow the `ProxyAdmin` to make a [Solidity interface call (high level call)](https://www.rareskills.io/post/low-level-call-solidity), it needs to accept the [ABI-encoded calldata](https://www.rareskills.io/post/abi-encoding) for `upgradeToAndCall()` from the `ProxyAdmin`.

Recall, the `ProxyAdmin` is making an interface call to `upgradeToAndCall` in the `Proxy` even though the proxy has no public functions besides the fallback (`ProxyAdmin` code show next):

![proxy admin code snippet with the upgradeToAndCall function highlighted](https://static.wixstatic.com/media/706568_02718aa0ba654f43a37caf29d743a7a2~mv2.png/v1/fill/w_740,h_477,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_02718aa0ba654f43a37caf29d743a7a2~mv2.png)

Below is a video showing all three code blocks side by side and how the different contracts in the inheritance chain (`Proxy`, `ERC1967Proxy`, and `TransparentUpgradeableProxy`) interact with each other:

<video src="https://video.wixstatic.com/video/706568_5ea96919975f40eea2ca1667881e32f6/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

## Why `upgradeToAndCall()` instead of just `upgradeTo()`?

When upgrading the implementation contract, it's possible to make a call to it as if the `ProxyAdmin` were `msg.sender` and have the transaction delegatecalled to the implementation as if it were a normal proxy interaction. Of course, this does not happen inside the fallback because calls from `ProxyAdmin` are routed to the upgrade logic.

The code below is from [ERC1967Utils.sol](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Utils.sol#L60C1-L76C6) which the `TransparentUpgradeableProxy` composes with to enable updating the implementation slot. The library provides an internal helper function to update the storage slot holding the implementation address.

```solidity
/**
* @dev Performs implementation upgrade with additional setup call if data is nonempty.
* This function is payable only if the setup call is performed, otherwise `msg.value` is rejected
* to avoid stuck value in the contract.
*
* Emits an {IERC1967-Upgraded} event.
*/

function upgradeToAndCall(address newImplementation, bytes memory data) internal {
    _setImplementation(newImplementation);
    emit IERC1967.Upgraded(newImplementation);
    if (data.length > 0) {
        Address.functionDelegateCall(newImplementation, data);
    } else {
        _checkNonPayable();
    }
}
```

It will only make a delegatecall to the implementation contract if `data.length > 0`.

`upgradeToAndCall()` also makes a delegatecall from the `Proxy` to the implementation in the same transaction as the upgrade. This is the same as if `ProxyAdmin` called the proxy using whatever calldata is specified in `data`, and then the proxy made a delegatecall to the implementation.

In this manner, the `ProxyAdmin` can make arbitrary calls to the `Proxy`.

**Note that upgradeToAndCall does not require the upgraded contract to be a different implementation — it is possible to "upgrade" to the same implementation.**

The implication of this is that the `ProxyAdmin` contract can make arbitrary delegatecalls to the implementation contract via the `Proxy` — but the `msg.sender` from the Transparent Proxy's perspective is the `ProxyAdmin`.

This is not a "problem" that the `ProxyAdmin` can use the contract — the `ProxyAdmin` has the ability to completely change the implementation — the owner of the `ProxyAdmin` already has admin control over the Proxy.

The only restriction the `ProxyAdmin` has on upgrading is that they cannot upgrade to an empty contract (an address with no bytecode). The `_setImplementation` function checks if the [code length](https://www.rareskills.io/post/solidity-code-length) of the new implementation is greater than zero

```solidity
/**
 * @dev Stores a new address in the ERC-1967 implementation slot.
 */
function _setImplementation(address newImplementation) private {
    if (newImplementation.code.length == 0) {
        revert ERC1967InvalidImplementation(newImplementation);
    }
    StorageSlot.getAddressSlot(IMPLEMENTATION_SLOT).value = newImplementation;
}
```

## Summary of Transparent Upgradeable Proxy

- The Transparent Upgradeable Proxy is a design pattern to prevent function selector clashing between the proxy and the implementation.
- The fallback function is the only public function on the Transparent Upgradeable Proxy.
- The upgrade functionality can only be invoked by the admin via the fallback function. All calls from non-admin addresses turn into delegatecalls to the proxy.
- The Transparent Upgradeable Proxy uses an immutable variable to store the admin to save gas. To be compliant with ERC-1967, it stores the address of the admin in the `admin` slot specified by ERC-1967, even though it never reads from that slot.
- Because the admin cannot be changed, the admin is set to be a smart contract called the `AdminProxy`. The `AdminProxy` exposes a single function `upgradeAndCall()` which can only be called by the owner of the `AdminProxy`. The owner of the `AdminProxy` can be changed. Such a change alters who can update the implementation slot in the Transparent Upgradeable Proxy.

*We would like to thank [@ernestognw](https://x.com/ernestognw) from OpenZeppelin for reviewing this article and providing helpful suggestions.*

*Originally Published Jun 4*
