# The Beacon Proxy Pattern Explained

![Beacon Proxy Pattern Banner by RareSkills](https://static.wixstatic.com/media/706568_32adfd3fb5e643bc835fd92d572f75b7~mv2.jpg/v1/fill/w_740,h_416,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/706568_32adfd3fb5e643bc835fd92d572f75b7~mv2.jpg)

A Beacon Proxy is a smart contract upgrade pattern where multiple proxies use the same implementation contract, and all the proxies can be upgraded in a single transaction. This article explains how this proxy pattern works.

## Prerequisites

We are going to assume that you already know how a [minimal proxy](https://www.rareskills.io/post/eip-1167-minimal-proxy-standard-with-initialization-clone-pattern) works and maybe even UUPS or [Transparent](https://www.rareskills.io/post/transparent-upgradeable-proxy).

## Motivation for Beacon Proxies

Typically, a proxy pattern uses a single implementation contract and a single proxy contract. However, it is possible for multiple proxies to use the same implementation.

![Multiple proxies pointing to a single implementation flowchart](https://static.wixstatic.com/media/706568_6faa10f3a0cf4f5a915778b010187b80~mv2.png/v1/fill/w_740,h_494,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_6faa10f3a0cf4f5a915778b010187b80~mv2.png)

To understand why we would want this, let's imagine a fully on-chain game. This game wants to store each user account as a separate contract, so that accounts can be easily transferred to different wallets and one wallet can own multiple accounts. Each proxy stores account information in its respective storage variables.

There are a couple of ways you could implement this:

1. Use Minimal Proxy Standard (EIP1167) and deploy each account as a clone
2. Use UUPS or Transparent proxy patterns and deploy a proxy for each account

In most cases either option would work but what if you wanted to add new functionality to the account?

In the case of the minimal proxy standard, you would have to redeploy the whole system and migrate everyone socially because the clones are not upgradeable.

Traditional proxies are upgradeable but you would have to upgrade each proxy one by one. This would be costly with more accounts.

Both clones and proxies are a hassle to upgrade when there are a lot of them.

**The beacon pattern is designed to solve this issue: It allows you to deploy a new implementation contract and upgrade all proxies simultaneously.**

This means the beacon pattern would allow you to deploy a new implementation of the account and upgrade all the proxies at once.

From a high level, this standard allows you to create an unlimited amount of proxies per implementation contract, and still be able to easily upgrade.

## How a beacon works

Like the name suggests, this standard requires a beacon, which OpenZeppelin refers to as "UpgradeableBeacon" and implements in `UpgradeableBeacon.sol`.

**The beacon is a smart contract that provides the current implementation address to the proxies via a public function.** The beacon is the source of truth for the proxies regarding the current implementation address, which is why it is called a "beacon".

When a proxy receives an incoming transaction, the proxy first calls the `view` function `implementation()` on the beacon to fetch the current implementation address, and then the proxy [delegatecalls](https://www.rareskills.io/post/delegatecall) to that address. This is what allows the beacon to function as the source of truth to where the implementation is.

![Beacon Proxy  step-by-step  delegatecall architecture](https://static.wixstatic.com/media/706568_8eb5ae7b2db340b78b743dfcf67ce4e4~mv2.png/v1/fill/w_740,h_494,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_8eb5ae7b2db340b78b743dfcf67ce4e4~mv2.png)

Any additional proxies will follow the same pattern: they first obtain the implementation address from the beacon using `implementation()` and then `delegatecall` to that address.

Note: the proxies know where to call `implementation()` because they store the beacon's address in an immutable variable. We will explain this mechanism more later.

This pattern is highly scalable because each additional proxy simply reads the implementation address from the beacon and then uses `delegatecall`.

![Beacon proxy getImplementation() function visualized](https://static.wixstatic.com/media/706568_b1b5d3eb8b504f5fa8abc45165c00982~mv2.png/v1/fill/w_740,h_494,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_b1b5d3eb8b504f5fa8abc45165c00982~mv2.png)

Although the beacon proxy pattern involves more contracts, the proxy itself is simpler than UUPS or Transparent Upgradeable Proxies.

The beacon proxies always call the same beacon address to get the current implementation address, so they don't need to concern themselves with details such as who the admin is or how to change the implementation address.

## Upgrading multiple proxies simultaneously

Since all the proxies get the implementation address from the beacon's storage, changing the address in the storage slot causes all the proxies to `delegatecall` to the new address, instantly "rerouting" them.

To upgrade all the proxies simultaneously:

1. Deploy a new implementation contract
2. Set the new implementation address in the beacon's storage
    
Setting the new implementation address is done by calling `upgradeTo(address newImplementation)` on the beacon and passing the new address as an argument. `upgradeTo()` is one of the two public functions on the `UpgradeableBeacon.sol` (the beacon). The other public (view) function is `implementation()` which we mentioned previously.

Note: [`upgradeTo()`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/05f218fb6617932e56bf5388c3b389c3028a7b73/contracts/proxy/beacon/UpgradeableBeacon.sol#L52C5-L55C1) has an `onlyOwner` modifier which is set in the constructor of `UpgradeableBeacon.sol` (the beacon).

```solidity
/**
 * @dev Upgrades the beacon to a new implementation.
 *
 * Emits an {Upgraded} event.
 *
 * Requirements:
 *
 * - msg.sender must be the owner of the contract.
 * - `newImplementation` must be a contract.
 */
function upgradeTo(address newImplementation) public virtual onlyOwner {
    _setImplementation(newImplementation);
}
```

`upgradeTo()` calls an internal function [`_setImplementation(address newImplementation)`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/05f218fb6617932e56bf5388c3b389c3028a7b73/contracts/proxy/beacon/UpgradeableBeacon.sol#L63) (also on the beacon), which checks if the new implementation address is a contract and then sets the address storage variable, `_implementation`, in the beacon to the new implementation address.

```solidity
/**
 * @dev Sets the implementation contract address for this beacon
 *
 * Requirements:
 *
 * - `newImplementation` must be a contract.
 */
function _setImplementation(address newImplementation) private {
    require(Address.isContract(newImplementation), "UpgradeableBeacon: implementation is not a contract");
    _implementation = newImplementation;
}
```

Now that the implementation address in the beacon's storage is changed, all the proxies will read the new address in the beacon and route their `delegatecall` to the new implementation.

This way of upgrading is simple because you are just "pointing" the beacon, and in turn the proxies, to a new implementation. You could even point the implementation back to a previous version if you needed to revert changes (be mindful of storage collisions).

![Upgrading a beacon proxy visualized](https://static.wixstatic.com/media/706568_8366adad15e94e65801484f0885364d1~mv2.png/v1/fill/w_740,h_494,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_8366adad15e94e65801484f0885364d1~mv2.png)

## Code walkthrough of the proxy contract

To avoid confusion, we use the terminology "BeaconProxy" to refer to the smart contract proxy and "beacon proxy" to refer to the design pattern. We will now discuss the proxy contract OpenZeppelin calls "BeaconProxy" and implements in `BeaconProxy.sol`.

The OpenZeppelin BeaconProxy inherits from `Proxy.sol` and adds more functionality:

1. It stores the address of the beacon contract in `_beacon`
2. A `_getBeacon()` function is added to return the `_beacon` variable
3. The `_implementation()` function is overridden to call `.implementation()` on the `_beacon` address
4. A constructor is added to set the `_beacon` variable and the `data` parameter initializes the proxy
    
Below is the OpenZeppelin implementation of a BeaconProxy with comments removed

```solidity
contract BeaconProxy is Proxy {
    address private immutable _beacon;

    constructor(address beacon, bytes memory data) payable {
        ERC1967Utils.upgradeBeaconToAndCall(beacon, data);
        _beacon = beacon;
    }

    function _implementation() internal view virtual override returns (address) {
        return IBeacon(_getBeacon()).implementation();
    }

    function _getBeacon() internal view virtual returns (address) {
        return _beacon;
    }
}
```

The `_implementation()` function is overridden because `Proxy.sol` calls that function to retrieve the implementation address before delegatecall.   

The constructor for the BeaconProxy serves two purposes:

1. set the `_beacon` address
2. initialize the proxy with `data`
    
This optional `data` is used in a `delegatecall` to the implementation, enabling initialization of the proxy's storage. In our game example, this could mean initializing the account (proxy) with the player's starting stats. Essentially, the data argument serves as the Solidity constructor for the proxy: the data is used in a `delegatecall` to the implementation so the implementation logic can configure the proxy storage variables.

```solidity
function upgradeBeaconToAndCall(address newBeacon, bytes memory data) internal {
    _setBeacon(newBeacon);
    emit IERC1967.BeaconUpgraded(newBeacon);

    if (data.length > 0) {
        Address.functionDelegateCall(IBeacon(newBeacon).implementation(), data);
    } else {
        _checkNonPayable();
    }
}
```

### ERC1967 & BeaconProxy.sol

For block explorers to know that the BeaconProxy is a proxy, it needs to adhere to the [ERC-1967](https://www.rareskills.io/post/erc1967) specification. Since it is specifically a beacon proxy, it needs to store the Beacon's address in the storage slot: `0xa3f0ad74e5423aebfd80d3ef4346578335a9a72aeaee59ff6cb3582b35133d50`, computed from `bytes32(uint256(keccak256('eip1967.proxy.beacon')) - 1)`.

Similar to the Transparent Upgradeable Proxy, this storage address is not actually used by the BeaconProxy. It is simply a signal to block explorers that the contract is a Beacon Proxy. The actual implementation address is stored in an immutable variable for [gas optimization](https://www.rareskills.io/post/gas-optimization) purposes; the beacon's address never changes.

### EIP2930

Always use [access list](https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum) transactions with this pattern, as you can save gas when doing a cross-contract call and accessing the storage of another contract. Specifically, the proxy is calling the beacon and getting the implementation address from storage. Access list benchmarking for a Beacon Proxy can be seen [here](https://github.com/RareSkills/access-list-benchmarks/tree/main/beacon_proxy).

## Deploying multiple BeaconProxies

Deploying several BeaconProxies manually would be a hassle. That's where the factory contract comes in. The factory deploys new proxies and sets the beacon address in their constructor.

OpenZeppelin does not require or provide a standard factory contract in their beacon pattern. However, in practice, a factory contract helps with deploying new proxies.

An example factory is provided below. The factory stores the address of the beacon and includes a function to create new proxies that use that beacon. The `createBeaconProxy()` function takes data as input to pass to the BeaconProxy's constructor. After deploying the proxy, it returns the proxy's address.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import {BeaconProxy} from "@openzeppelin/contracts/proxy/beacon/BeaconProxy.sol";

/// @dev THIS CONTRACT IS FOR TEACHING PURPOSES ONLY
contract ExampleFactory {

    address public immutable beacon;

    constructor(address Beacon) {
        beacon = Beacon;
    }

    function createBeaconProxy(bytes memory data) external returns (address) {
        BeaconProxy proxy = new BeaconProxy(beacon, data);
        return address(proxy);
    }
}
```

Now that we understand how to deploy proxies with a factory contract, let's see how it fits into the overall structure.

![Beacon proxy with Factory Contract simplified flowchart](https://static.wixstatic.com/media/706568_a1f8369c48294049871160064cb0ab3e~mv2.png/v1/fill/w_740,h_494,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_a1f8369c48294049871160064cb0ab3e~mv2.png)

And that's all the contracts needed to design a beacon pattern:

- Implementation
- Beacon
- Factory (optional)
- Proxy(s)

## Deploying

Ok but how do we deploy this whole system? It's not as scary as it may seem.

OpenZeppelin offers an [Upgrades](https://docs.openzeppelin.com/upgrades-plugins/1.x/) plugin for both Hardhat and [Foundry](https://www.rareskills.io/post/foundry-testing-solidity). It's as simple as installing the library and just calling `deployBeacon()` with the parameters for the beacon contract. From there, BeaconProxies can be deployed by calling `deployBeaconProxy()`. Upgrading is similar: the `upgradeBeacon()` function is called with the parameters for the new implementation.

The system can also be deployed manually:

1. Deploy the implementation contract
2. Deploy the beacon contract and in the constructor, input the implementation address and the address of who is allowed to upgrade the implementation address
3. Deploy the factory contract
4. Use the factory to deploy as many proxies as needed

![Order of deployment for Beacon Proxy with Factory](https://static.wixstatic.com/media/706568_b2aa7acb088b4efdad6c32aaf6a4195c~mv2.png/v1/fill/w_740,h_494,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_b2aa7acb088b4efdad6c32aaf6a4195c~mv2.png)

## A Real World Example

When would a beacon proxy be used in real life? I created a beacon proxy for Kwenta that's live on Optimism with 20M+ in TVL.

The beacon proxy was for Kwenta vesting packages. A "vesting package" is a smart contract that slowly releases tokens (KWENTA) to special interests and core contributors of the protocol. Each person is given a vesting package which varies in token amounts and duration (usually 1-4 years). To learn more about vesting in crypto see [here](https://cointelegraph.com/explained/vesting-in-crypto-explained).

Why a beacon proxy specifically?

1. It had to be easily upgradeable. Vesting packages had to be upgradeable because they call functions on the Kwenta staking system which is also upgradeable. If the staking system is upgraded in the future, then functionality on the vesting packages might no longer work. Making vesting packages upgradeable allows for them to be future-proof
2. Every package had the same vesting logic ( `vest()`, `stake()`, etc.) but different initialized parameters (token amounts, vesting lengths). Part of this required making vesting packages to be standalone contracts or "siloed" because

    a. Simpler development: having one initializable contract per person was a lot simpler than having one large contract with complex mappings to keep track of everyone's different vesting package. Also, the KWENTA for each package was automatically staked upon package creation which meant that each person was accruing rewards. If everyone's packages were all together in 1 contract then rewards would get intermingled and messy.

    b. Ownership of vesting packages could be easily transferred to other addresses or a multi-sig.

    c. Vesting meant calling `unstake()` on the Kwenta staking contract. The staking contract has a 2-week `unstake()` cooldown. So if everyone's packages were in one contract, and one person vested (and in turn unstaked) then no one else could vest for at least 2 weeks. Siloing the packages into separate contracts avoids this bug.
    
3. Vesting packages had to support 10+ people. This meant 10+ proxies

**A beacon proxy was able to do all of those without sacrificing anything.**

Clones could easily deploy 10+ initializable contracts but they aren't upgradeable.

Transparent and UUPS are upgradeable but would require each vesting package to be upgraded one by one which would've been time-consuming and cost more gas.

And a diamond proxy was considered but was too complex for this structure.

### Kwenta's FactoryBeacon

As an optimization, `FactoryBeacon` combines the `UpgradeableBeacon.sol` and `Factory` contracts. This combination simplifies the setup and reduces the surface area.

![Kwenta's FactoryBeacon Flowchart by RareSkills](https://static.wixstatic.com/media/706568_a59e9f2928d2446daa125a82c3e28372~mv2.png/v1/fill/w_740,h_494,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_a59e9f2928d2446daa125a82c3e28372~mv2.png)

This is possible because the factory doesn't need to be a standalone contract: It's just a few lines of code that deploy a new BeaconProxy and sets its beacon address and initialization data.

Here is an example of a combined factory and beacon contract. By inheriting `UpgradeableBeacon`, the contract retains the same functionality as a regular beacon, while the `createBeaconProxy()` function adds the factory functionality. Additionally, there is no longer a need to store the beacon address, as `address(this)` can now be used.

![Combined Factory and Beacon contract Code Snippet](https://static.wixstatic.com/media/706568_0c2b59ac16f94269b83b9f9debda75a6~mv2.png/v1/fill/w_740,h_250,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_0c2b59ac16f94269b83b9f9debda75a6~mv2.png)

**Nonetheless, the overall "beacon structure" is still the same.**

Each person calls their `BeaconProxy` which has all their storage for their specific vesting package (vesting amounts, duration).

The `BeaconProxy` then gets the implementation address from the `FactoryBeacon`, which still has the same functions as a regular beacon.

After getting the implementation address from the `FactoryBeacon`, the `BeaconProxy` then `delegatecalls` to the `VestingBaseV2` which is just the implementation.

Note that the only one who can call the `FactoryBeacon` is the adminDAO (an admin multisig). An admin is the only one who can create a new vesting package (`BeaconProxy`) and upgrade the proxies to a new implementation.

## Conclusion

The beacon proxy pattern allows the creation of multiple proxies for one implementation, with the ability to upgrade them all at once. The factory deploys new proxies, which use `delegatecall` to an address retrieved from the beacon. The beacon serves as the source of truth for the implementation.

It should be noted that the beacon proxy pattern incurs higher gas costs during setup compared to other patterns like UUPS or Transparent because both a factory and a beacon must be deployed in addition to the proxy. Additionally, every call to a proxy incurs an additional cost to call the beacon. This extra gas cost is the main drawback. It isn't necessarily a disadvantage if you need multiple proxies, as this is when the beacon proxy pattern is most beneficial. The higher gas cost is why you won't typically see a beacon proxy pattern used with only one proxy.

While beacons allow for upgrading multiple proxies simultaneously, the setup is more complex and costly. It requires more gas and involves setting up additional contracts, making it more expensive in terms of development and auditing. Therefore, the beacon proxy pattern is advantageous only if you need a large number of proxies.

## Authorship

This article was written by Andrew Chiaramonte ([LinkedIn](https://www.linkedin.com/in/andrewcmonte/), [Twitter](https://x.com/andrewcmonte)).

*Originally Published July 20, 2024*
