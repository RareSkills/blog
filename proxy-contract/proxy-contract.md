# Introduction to Proxies

Proxy contracts enable smart contracts to retain their state while allowing their logic to be upgraded.

By default, smart contracts cannot be upgraded because deployed bytecode cannot be modified.

The only mechanism in the EVM to change bytecode is to deploy a new contract. However, the storage in this new contract would not "know anything" about the previous contract. This means previous values kept in storage would be unavailable to the new contract.

The solution that proxies introduce is to keep the storage in one contract, and get the business logic and functionality (provided by the bytecode) from another contract. If new functionality is needed, then a new "logic contract" is deployed, but the storage contract remains the same.

When a call is made to the storage contract, the storage contract simply [delegatecalls](https://www.rareskills.io/post/delegatecall) the function in the logic contract to make the state update.

*To understand proxies, understanding [delegatecall](https://www.rareskills.io/post/delegatecall) is crucial, so please read the linked article first.*

What we are calling the "storage contract" here is the proxy contract.

In this article, we will teach:

1. How proxies work and how to create them 
2. How to upgrade a proxy smart contract

***Disclaimer*:** The proxy implementation demonstrated here is for learning purposes only and should not be used in production. For production-level proxies, please see the later chapters in our book on [Proxy Patterns](https://www.rareskills.io/proxy-patterns). But you should read this chapter first to have a better foundation before reading the later chapters.

## What is a proxy contract?

A proxy contract is a smart contract that stores state variables while delegating all its logic to one or several implementation contracts. That is, a proxy contract merely keeps the storage variables, while a separate contract’s logic updates the storage variables.

You can think of a proxy contract as similar to apps and data on your mobile phone. The phone retains your personal data—contacts, photos, and browsing history—just as the proxy retains its state. The implementation contract is like the phone's operating system (OS) and apps, responsible for its functionality and behavior. When the OS or apps are updated, the phone's features improve, but your data remains intact.

<video src="https://r2media.rareskills.io/Intro-Proxies/UpgradePhoneCompressed.mp4" type="video/mp4" autoplay loop muted controls></video>

This analogy illustrates how a proxy contract keeps its state while delegating functionality to an implementation contract.

![a diagram showing how a proxy contract keeps its state while delegating functionality to an implementation contract](https://r2media.rareskills.io/Intro-Proxies/delegatecallDiagram.png)

A proxy contract and its logic contract are set up as follows:

1. You deploy a proxy contract
2. You then deploy the implementation contract
3. You store the implementation contract address in the proxy's storage
4. Now, the proxy forwards all calls to the implementation address through `DELEGATECALL` 

## How are calls forwarded?

Since the proxy has no logic of its own, any calls made to the proxy contract will be caught by the [`fallback`](https://www.rareskills.io/post/fallback-extension-pattern)  function. The fallback function handles cases where a function call does not match any defined functions in a contract.

The proxy will then `delegatecall` the implementation using the same `calldata` the proxy received as illustrated in the diagram below:

![a diagram showing how calldata is forwarded via delegatecall](https://r2media.rareskills.io/Intro-Proxies/calldataToFallback.png)

The proxy contract always `delegatecalls`to the implementation using the same [`calldata`](https://www.rareskills.io/post/function-selector) it received.

Here is an animation from [our article on delegatecall](https://www.rareskills.io/post/delegatecall) as a review:

<video src="https://r2media.rareskills.io/Intro-Proxies/delegatecallAnimation.mp4" type="video/mp4" autoplay loop muted controls></video> 

## Why proxy contracts?

Proxy contracts have two notable use cases:

### 1. Upgradeability

The ability to upgrade a contract is the most common use case of proxy contracts. A proxy pattern allows you to create contracts that can be upgraded (incorporate new logic or features) without disrupting the existing contract's state or address.

On the other hand, a non-upgradeable contract requires you to convince all users, wallet providers, and exchanges to migrate to a smart contract address every time you fix a bug or add new features.

### 2. Saving deployment gas cost

Proxies can [save gas](https://www.rareskills.io/post/smart-contract-creation-cost) if multiple copies of a contract need to be deployed, as all the proxies can use the same logic contract while keeping their own separate state. Instead of deploying full contract logic for every copy, you deploy a single implementation contract, and all the proxies use `delegatecall` to interact with it. Since the logic is very simple, the deployed bytecode is a lot smaller and thus cheaper to deploy. This pattern is called the [Minimal Proxy Pattern](https://www.rareskills.io/post/eip-1167-minimal-proxy-standard-with-initialization-clone-pattern) (more about this later in the article).

## How to deploy an upgradeable smart contract (not for production)

Let's start by creating a simple proxy contract that `delegatecalls` a single hardcoded implementation address. This will help us understand the basic pattern before making it upgradeable.

### **1.** Deploy your implementation contract

Let’s consider the contract below as our implementation contract. It takes two numbers as arguments and emits their sum as an [event](https://www.rareskills.io/post/ethereum-events).

```solidity
contract Implementation {
    event Result(uint256 newValue);

    function addNumbers(uint256 number1, uint256 number2) public returns (uint256 result ) {
        result = number1 + number2;
        emit Result(result);
    }
}
```

We deploy the implementation contract using Remix as shown below. After deployment, we copy the address:

![A diagram showing how to deploy the implementation contract using Remix](https://r2media.rareskills.io/Intro-Proxies/proxiesDrawioGm.png)

### **2. Deploy a proxy contract and set the implementation address**

We can now deploy the `Proxy` contract with the `Implementation` contract address stored in the `Proxy` contract.

```solidity
                                 // Replace this with the implementation address
                                 // you get when you deployed the implementation 
                                 // contract
address immutable implementation = 0x44fE319Fc0C2d3e09F9a86AF94eEabB6173A1d58;
```

Below we show an example of a proxy contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

contract Proxy {
    // Change the implementation address to the one you get after deploying the 
    // implementation contract
    address immutable implementation = 0x44fE319Fc0C2d3e09F9a86AF94eEabB6173A1d58;

    fallback(bytes calldata data) external returns (bytes memory) {
        (bool success, bytes memory result) = implementation.delegatecall(data);
        require(success, "Delegatecall failed");
        return result;
     }
}
```

Deploy the proxy contract on Remix following the illustration in the diagram below.

![A diagram showing how to deploy the proxy contract and set the implementation address](https://r2media.rareskills.io/Intro-Proxies/deployProxyCodeScreenshot.png)

### Testing/Interacting with your contract

When the contract deployment is completed, the next step would be interacting with the proxy contract. We’ll explore two approaches to interacting with your contract.

### 1. Interacting with the proxy contract using the `Low Level Interaction` interface in Remix

The first way to interact with the proxy contract in Remix is by using the `Low Level Interaction` interface. This involves constructing a `calldata` for our target function and passing it into `calldata` input box.

So, to trigger the `addNumbers` function to add two numbers `5` and `4` we’ll construct the `calldata` by performing the [ABI encoding](https://www.rareskills.io/post/abi-encoding) of the function and its parameters as shown below:

```solidity
function seeEncoding() external pure returns (bytes memory) {
    return abi.encodeWithSignature("addNumbers(uint256,uint256)", 5,4);
}
```

The result will be the code below:

```solidity
0xef9fc50b00000000000000000000000000000000000000000000000000000000000000050000000000000000000000000000000000000000000000000000000000000004
```

Now that we’ve constructed the `calldata` let’s use it to call the function in the proxy contract.

Paste the `calldata` to the input box. We should expect the result to be 9 since 5+4 is 9 as shown in the screenshot below:

![A diagram showing how to use the calldata to call a function in the proxy contract in Remix](https://r2media.rareskills.io/Intro-Proxies/proxiesDrawioAgain.png)

Even though the Proxy contract has no logic to emit events, we still see an event emitted when we sent the `calldata` to the proxy. That is because the event emitting logic was delegatecalled by the proxy.

Clearly this process looks a little complicated to test your contracts, especially if we have to manually encode your own `calldata`. A more simpler alternative is to interact with the `Proxy` contract using the `Implementation` contract’s ABI.

### 2. Interacting with the `Proxy` contract using the `Implementation` contracts’s ABI in Remix

Follow these steps after deployment to setup the proxy contract so you can interact with it via the implementation contract’s ABI: 

1.  In the `CONTRACT` dropdown, select the `Implementation` contract. 

![A diagram showing how to select the Implementation contract on Remix](https://r2media.rareskills.io/Intro-Proxies/selectImplementationRemix.png)

2. Copy the `Proxy` contract address and paste it into the `At Address` input box.

![A diagram showing how to copy the Proxy contract address and paste it into the `At Address` input box](https://r2media.rareskills.io/Intro-Proxies/pasteAddressScreenshot.png)

3. Click the `At Address` button to interact with the contract, as shown in the diagram below.

![A diagram showing a click on the At Address and an interaction with the proxy contract in Remix](https://r2media.rareskills.io/Intro-Proxies/atAddressScreenshot.png)

Now, you should be able to interact with the `addNumbers` function as shown in the diagram below:

![A diagram showing an interaction with the proxy contract](https://r2media.rareskills.io/Intro-Proxies/addNumbersFnCallRemix.png)

*Note: even though Remix labeled the contract we are interacting with as `Implementation`, it is actually the `Proxy` contract using the `Implementation` contracts’s ABI.*

As we can see, when the user interacts with the Proxy contract and attempts to call the `addNumbers` function, it triggers the `fallback` function because the `addNumbers` function does not exist in the Proxy. Once triggered, the `fallback` function forwards the execution to the Implementation contract using `delegatecall`, where the function is defined.

The screen recording below summarizes the above steps:

<video src="https://r2media.rareskills.io/Intro-Proxies/proxyDemo.mp4" type="video/mp4" autoplay loop muted controls></video>


So far, we've seen a basic proxy contract that delegates calls to a hardcoded implementation address. However, this approach is not upgradeable, as the implementation address is fixed in the contract bytecode because of the immutable keyword.

```solidity
address immutable implementation = 0x44fE319Fc0C2d3e09F9a86AF94eEabB6173A1d58;
```

### Update proxy’s implementation

To make the proxy contract upgradeable, we need to store the implementation address in a way that can be updated after deployment. We can do this by adding a `setImplementation` function to the proxy contract:

```solidity
function setImplementation(address _implementation) public onlyOwner {
    implementation = _implementation;
}
```

Now, instead of hardcoding the implementation address, we store it in a state variable `implementation` . That way, we can update it when necessary.

```solidity
contract Proxy {
    // Store the implementation contract address
    address implementation;

    function setImplementation(address _implementation) public {
        implementation = _implementation;
    }

    fallback(bytes calldata data) external returns (bytes memory) {
        (bool success, bytes memory result) = implementation.delegatecall(data);
        require(success, "Delegatecall failed");
        return result;
     }
}
```

For security purposes, we need to ensure that only an admin can update the implementation address. We achieve this by introducing an `admin` state variable: 

```solidity
contract Proxy {
    address public implementation;
    address public admin;
    ...
```

And restricting access to the `setImplementation` function with a modifier:

```solidity
modifier onlyOwner() {
   require(msg.sender == admin, "Not the contract owner");
   _;
}
```

## Storage collision problem

Now our proxy contract includes the `implementation`, `admin`, and `number` (we’ve introduced the `number` to illustrate the storage collision problem) storage variables and the storage layout now looks like this:

```solidity
contract Proxy {
    address public implementation;
    address public admin; 
    uint256 public number;
    ...
```

And in our implementation contract, we have the `number` state variable:

```solidity
contract Implementation {
    uint256 public number;
    ...
```

This will result in a storage collision. When we try to update the `number` state variable in the proxy contract, we’ll be overwriting the `implementation` address state which is not what we want!

In Solidity, storage variables are assigned to fixed [storage slots](https://www.rareskills.io/post/evm-solidity-storage-layout) based on their order of declaration in the contract. This means that if the layout of the storage variables in the proxy and the implementation doesn’t match, conflicts will happen.

In the our case:

- The proxy contract stores the `implementation` address at slot 0, `admin` at slot 1, and `number` at slot 2.
- In the implementation contract, the `number` variable is stored in slot 0.

This causes a conflict because when the implementation contract tries to update its `number` variable, it ends up modifying the `implementation` address in the proxy contract instead, which is stored at the same slot (slot 0). 

![A diagram showing a storage overlap](https://r2media.rareskills.io/Intro-Proxies/slot0Overlap.png)

This leads to an unexpected behavior—you want to update the `number` in the proxy, but you’re actually overwriting the `implementation` address in the storage.

*Our articles [Storage Slots in Solidity: Storage Allocation and Low-level assembly storage operations](https://www.rareskills.io/post/evm-solidity-storage-layout) and [Storage Slot III (Complex Types)](https://www.rareskills.io/post/solidity-dynamic) explains in detail how storage slots work in Solidity with useful diagrams and animations.*

Let’s use the following contract as an example:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

contract Proxy {
    address public implementation;
    address public admin;
    uint256 public number;

    constructor() {
        admin = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == admin, "Not the contract owner");
        _;
    }

    function setImplementation(address _implementation) public onlyOwner {
        implementation = _implementation;
    }

    fallback(bytes calldata data) external returns (bytes memory) {
        require(implementation != address(0), "Implementation not set");

        (bool success, bytes memory result) = implementation.delegatecall(data);
        require(success, "Delegatecall failed");
        return result;
     }
}

contract Implementation {
    uint256 public number;

    function increment() public {
        number++; 
    }
}

```

You’ll notice that the increment will not work correctly because it’s updating the implementation contract’s address instead of `number` in storage:

![A diagram showing a return value of 0 from the contract interaction](https://r2media.rareskills.io/Intro-Proxies/returned0.png)

### How do we solve this storage collision problem in proxies?

One way is to pick a random slot such that a collision is extremely unlikely. More details about how we pick such a slot are in [ERC-1967](https://www.rareskills.io/post/erc1967).

So, following the ERC-1967 convention, we’ll use the slot below to store the `implementation` address:

```solidity
0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc
```

The slot is derived pseudorandomly from:

```solidity
bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)
```

And we’ll use the slot below to store the `admin` address:

```solidity
0xb53127684a568b3173ae13b9f8a6016e243e63b6e8ee1178d6a717850b5d6103
```

Derived from:

```solidity
bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1)
```

To read or write to these specific storage slots, you'll need to use `sload` and `sstore` respectively in inline assembly.

The code below is a revised version of our initial proxy contract, now using storage slots defined by the ERC-1967 standard to ensure no storage collisions between the proxy and implementation contracts.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

contract Proxy {

    /**
     * @dev Storage slot for the implementation address.
     * This is derived from  `bytes32(uint256(keccak256('eip1967.proxy.implementation')) - 1)`.
     */
    bytes32 private constant _IMPLEMENTATION_SLOT = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;

    /**
     * @dev Storage slot for the admin address.
     * This is derived from `bytes32(uint256(keccak256('eip1967.proxy.admin')) - 1)`.
     */
    bytes32 private constant _ADMIN_SLOT = 0xb53127684a568b3173ae13b9f8a6016eaf15eb9e8e9f03347e2db6a3ec1e1cb0;

		// Initialize proxy with the owner
    constructor() {
        address admin = msg.sender;
        assembly {
            // Store admin in the ERC-1967 admin slot
            sstore(_ADMIN_SLOT, admin)
        }
    }

    modifier onlyOwner() {
        address admin;
        assembly {
            // Load admin from the ERC-1967 admin slot
            admin := sload(_ADMIN_SLOT)
        }
        require(msg.sender == admin, "Not the contract owner");
        _;
    }

    function setImplementation(address _implementation) public onlyOwner {
        assembly {
            // Store implementation in the ERC-1967 implementation slot
            sstore(_IMPLEMENTATION_SLOT, _implementation)
        }
    }

    fallback(bytes calldata data) external payable returns (bytes memory) {
        address implementation;
        assembly {
            // Load implementation from the ERC-1967 implementation slot
            implementation := sload(_IMPLEMENTATION_SLOT)
        }
        require(implementation != address(0), "Implementation not set");

        (bool success, bytes memory result) = implementation.delegatecall(data);
        require(success, "Delegatecall failed");
        return result;
    }
}

contract Implementation {
    uint256 public number;

    function increment() public {
        number++; 
    }
}
```

Now, if we deploy this contract and run it, we’ll get the desired result as shown below:

![A diagram showing a return value of 2 from the contract interaction](https://r2media.rareskills.io/Intro-Proxies/returned2.png)

We discuss EIP-1967 extensively in our [Storage Slots for Proxies](https://www.rareskills.io/post/erc1967) article, and it is the next chapter in this book.

## Conclusion

In this article, we've explored the concept of proxy contracts, their importance, and how they enable the upgradeability and reduce Solidity smart contracts deployment costs.

Here are some key takeaways from this piece:

- The [`DELEGATECALL`](https://www.rareskills.io/post/delegatecall) opcode makes upgradeability possible with proxies.
- Proxy contracts (both upgradeable and non-upgradeable) holds the storage, including the implementation address. The implementation contracts holds the logic.
- Upgradeable Proxy holds the storage, the implementation address, and provides a setter function to update the implementation address. The implementation holds only the logic

Further reading recommendation:

- https://www.rareskills.io/proxy-patterns
