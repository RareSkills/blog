# The initializable smart contract design pattern

Initializers are how upgradeable contracts achieve the behavior of a constructor.

When deploying contracts, it's common to call a constructor to initialize storage variables. For example, a constructor might set the name of a token or the maximum supply. In an upgradeable contract, this information needs to be stored in the proxy's storage variables, not the implementation's storage variables. Adding a constructor in the proxy such as:

```solidity
constructor(
    string memory name_,
    string memory symbol_,
    uint256 maxSupply_
) {
    name = name_;
    symbol = symbol_;
    maxSupply = maxSupply_;
}
```

is not a good solution because it is error prone to align the storage variable locations between the proxy and the implementation. Creating a constructor in the implementation won't work because it will set the storage variables in the implementation.

## How initializers work

The solution to all of the problems above is to create an `initializer()` function in the implementation which sets the storage variables in the same manner a constructor would and have the proxy [delegatecall](https://www.rareskills.io/post/delegatecall) `initialize()` the implementation. Putting the `initializer()` in the implementation ensures that the storage variable alignment will be automatically correct. **To mimic a constructor, it is crucial that this function can only be delegatecalled once by the proxy.**

The purpose of OpenZeppelin's [`Initializable.sol`](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/utils/Initializable.sol) contract is to provide a robust implementation of this initialization pattern. Currently, `Initializable.sol` is used in OpenZeppelin's upgradable contracts, such as `ERC20Upgradeable.sol`.

The purpose of this article is to provide a detailed explanation of how Initializable.sol works. But before that, let's show how this pattern could be implemented naïvely and why the naïve implementation only works in the simplest scenarios.

A high level illustration of the initializer process is shown in the animation below:

<video src="https://video.wixstatic.com/video/706568_60bc2f10e4d54c2194771dae0df99a37/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls>
</video>

## A naïve Implementation

It may be tempting to write a contract like the one below, where a modifier is devised to restrict a function's execution to just once and never again.

```solidity
contract NaiveInitialization {
    // initialized indicates if the contract has been initialized
    bool initialized = false;

    // restricts the function to be executed only once
    modifier initializer() {
        require(initialized == false, "Already initialized");
        initialized = true;
        _;
    }

    // can be executed only once
    function initialize() public initializer {
        // Initialize necessary storage variables
    }
}
```

The above code works for this specific contract, ensuring that the `initialize()` function can only be executed once. However, the same pattern fails when used with inheritance.

### Failed Implementation Demo of Initialize

The problem with the pattern above is that it doesn't support when contracts use inheritance and parent contracts also have to be initialized. Let's examine an example of this issue in the following code.

```solidity
contract Initializable {
    // initialized indicates if the contract has been initialized
    bool initialized = false;

    // restricts the function to be executed only once
    modifier initializer() {
        require(initialized == false, "Already initialized");
        initialized = true;
        _;
    }
}

contract ParentNaive is Initializable {
    // initialize the Parent contract
    function initializeParent() internal initializer {
        // Initialize some state variables
    }
}

contract ChildNaive is ParentNaive {
    // initialize the Child contract
    function initializeChild() public initializer {
        super.initializeParent();
        // Initialize other state variables
    }
}
```

The expected execution order of the above contract is as follows:

1. The `initializeChild()` function is called. With the `initializer` modifier, it updates the `initialized` variable to `true`. This variable is used by *all* contracts in the inheritance chain.
2. Next, the `initializeParent()` function is called within `initializeChild()`. `initializeParent()` also has the `initializer` modifier, so it requires the variable `initialized` to be `false`.
3. **But the** `initialized` **variable has already been set to** `true` when `initializeChild` **ran, so the transaction will revert when** `initializeParent()` **is called**.

OpenZeppelin's `Initializable.sol` contract addresses this issue by allowing initialization for all contracts within an inheritance chain, while still preventing initializers from being called after the initialization transaction.

## Understanding Initializable.sol

The core of `Initializable.sol` consists of three modifiers: `initializer`, `reinitializer` and `onlyInitializing`, and two state variables: `_initializing` and `_initialized`.

Each modifier is only used in a specific scenario and serves a distinct purpose. An overview of their use is as follows:

- The `initializer` modifier should be used during the initial deployment of the upgradable contract and exclusively in the childmost contract.
- The `reinitializer` modifier should be used to initialize new versions of the implementation contract, again only within childmost contracts.
- The `onlyInitializing` modifier is used with parent initializers to run during initialization and prevents those initializers from being called in a later transaction. This solves the problem mentioned in the previous section, where the parent initializers could not run due to the childmost initializer disabling them. With this scheme, it is possible to initialize all parent contracts, as well as the childmost contract.
    
Below is a visual diagram illustrating these scenarios. A more detailed explanation of the use of these modifiers will be provided in the following sections.

![Visual diagram showing the purpose of the three core Initializable.sol modifiers: initializer, reinitializer and onlyInitializing.](https://static.wixstatic.com/media/706568_805b0817d9cf42768bf504d5a5b4553a~mv2.png/v1/fill/w_719,h_428,al_c,lg_1,q_85,enc_auto/706568_805b0817d9cf42768bf504d5a5b4553a~mv2.png)

**Initializable.sol** implements the [ERC-7201 pattern](https://www.rareskills.io/post/erc-7201), where state variables are declared within a struct. If you are not aware of this pattern, simply consider `_initializing` and `_initialized` as state variables.

```solidity
struct InitializableStorage {
    /**
     * @dev Indicates that the contract has been initialized.
     */
    uint64 _initialized;

    /**
     * @dev Indicates that the contract is in the process of being initialized.
     */
    bool _initializing;
}
```

The `_initializing` variable is a boolean that indicates whether the contract is in the process of initialization, while the `_initialized` variable stores the current version of the contract. It starts at value `0` and will be `1` after the first initialization. It could be higher if the developers chose to deploy a new implementation and wish to "reinitialize" the storage variables to new values.

The video below illustrates how these parts work together:

<video src="https://video.wixstatic.com/video/706568_ed24969c5dea40ee9d762eeeebc61c78/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls>
</video>

## The Initializer modifier

The `initializer` modifier is as follows. Some parts of the code will be explained in more detail later.

![Code snippet of the initializable.sol initializer modifier](https://static.wixstatic.com/media/706568_77b7b566ccf3432889f75a0ea986b382~mv2.png/v1/fill/w_740,h_630,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_77b7b566ccf3432889f75a0ea986b382~mv2.png)

The code above is not straightforward due to the necessity to address backward compatibility issues with previous versions. However, the main idea is twofold:  
1. Set the `_initialized` variable to `1` to prevent the function from being executed again (<span style="color:Green">green box</span>).
2. Temporarily allow the parent initializers, modified with `onlyInitializing`, to run while `_initializing` is true. As can be seen in the code above, `_initializing` is false when the contract has not been initialized, true while the initialization transaction is running, and false when the initialization transaction finishes.
    
Because `initializer` requires that `_initializing` be false, it cannot be used in parent contracts within the inheritance chain, since `_initializing` is true while these are executing. Instead, initialization functions of parent contracts must use a different modifier, specifically `onlyInitializing`, which allows the function to execute only when `_initializing` is true.

## The onlyInitializing modifier

The `onlyInitializing` modifier is designed for use in parent contracts, as it only executes while `_initializing` is true, as shown below.

```solidity
modifier onlyInitializing() {
    _checkInitializing();
    _;
}

function _checkInitializing() internal view virtual {
    if (!_isInitializing()) {
        revert NotInitializing(); // reverts if _initializing is false
    }
}
```

Below is a visual representation of this flow.

![Visual representation of the onlyInitializing modifier flow](https://static.wixstatic.com/media/706568_4bd611d6b66e48b7bfbea85a1d08ef9f~mv2.png/v1/fill/w_697,h_622,al_c,lg_1,q_90,enc_auto/706568_4bd611d6b66e48b7bfbea85a1d08ef9f~mv2.png)

To summarize, parent initializers are protected by the `onlyInitializing` modifier which prevents them from being called unless the childmost contract's `inititializer` is currently executing.

## The reinitializer modifier

The `reinitializer` modifier serves a similar role to `initializer`, but must be used to initialize new versions of the implementation contract *if* a new version needs to update the storage variables upon initialization.

This modifier has a `uint64` argument that indicates a contract version, which must be greater than the current version. If one wants to prevent future reinitialization, the version can be set to $2^{64} − 1$ or `type(uint64).max`. Making the variable `uint64` allows the `_initializing` boolean variable to be packed in the same slot, and $2^{64} − 1$ versions leaves plenty of room for future upgrades. Here is the code for the `reinitializer` modifier.

```solidity
modifier reinitializer(uint64 version) {
    // solhint-disable-next-line var-name-mixedcase
    InitializableStorage storage $ = _getInitializableStorage();

    if ($._initializing || $._initialized >= version) {
        revert InvalidInitialization();
    }
    $._initialized = version;
    $._initializing = true;
    _;
    $._initializing = false;

    emit Initialized(version);
}
```

Let's illustrate its use with an example. Suppose in the first upgrade of an **ERC20Upgradeable** contract, we want to change the name and symbol of the token. This function must be written as follows, where the value `2` indicates that this is the second version of the contract:

```solidity
function  initialize() reinitializer(2) public {
    __ERC20_init("MyToken2", "MTK2");
}
```

To summarize how to proceed with upgrades: 

1. You can't use `initializer` on a new version.
2. You must use `reinitializer` *if* you want to change any state variables in an initialization function.
3. Alternatively, you could have no initializer if you don't need to make state changes during the upgrade.

## Uninitialized contracts vulnerability

In the modifier `initializer`, there is a line of code that perhaps caught your attention, but I didn't mention it in my initial explanation. It is as follows:

```solidity
bool construction = initialized == 1 && address(this).code.length == 0;
```

The expression `address(this).code.length == 0` is true only during contract deployment. Therefore, the `construction` variable can only be true if the `initializer` modifier is being used in a constructor.

In fact, the `initializer` modifier can be used in the constructor of the implementation contract to "initialize" it. This might seem counterintuitive since the storage of the implementation contract should not matter. However, as we will see, this initialization serves as a security measure.

### The UUPS vulnerability involving uninitialized contracts.

The crucial point is that the initialization function of a contract is public and can be called either through the proxy or directly from an EOA or another contract, as illustrated in the figure below. In [upgradeable `Ownable` contracts](https://www.rareskills.io/post/openzeppelin-ownable2step), the owner is typically set in the initialize function.

- If `initialize` is called via delegatecall from the proxy, the owner is stored in the proxy's storage.
- If `initialize` is called *directly* on the implementation, the owner is stored in the implementation's storage.

![Owner in proxy storage and implementation storage calling the initialize function in the proxy and implementation respectively](https://static.wixstatic.com/media/706568_ca39f40c6cc34b8a967401a73e0019fe~mv2.png/v1/fill/w_738,h_402,al_c,lg_1,q_85,enc_auto/706568_ca39f40c6cc34b8a967401a73e0019fe~mv2.png)

Being the owner of the implementation contract should not matter, as executing functions directly on the implementation contract modifies its own storage, which is not the 'real' storage. For this reason, many teams didn't consider executing the initialize function directly on the implementation contract.

The critical issue is that any function defined as `onlyOwner` can then be executed by this 'other' owner on the implementation contract.

And it was precisely this scenario that exposed a vulnerability in OpenZeppelin's UUPS contracts from v4.1.0 through v4.3.1. The function responsible for migrating from one implementation contract to the next also executed a delegatecall to that new address.

![Open Zeppelin's UUPS upgradeToAndCall function code snippet with a red box highlighting the if(data.length>0) conditional statement](https://static.wixstatic.com/media/706568_ec1c9915f81e479f8ff924802f654409~mv2.png/v1/fill/w_740,h_309,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_ec1c9915f81e479f8ff924802f654409~mv2.png)

This function was protected by an `onlyOwner` modifier and intended to be executed only by the "rightful owner". However, it could also be executed by the owner of the implementation contract.

Modifying the implementation's storage wasn't the issue. However, the owner could now delegatecall to a contract containing a selfdestruct opcode. This action would erase the implementation contract's code, preventing the proxy from migrating to a new implementation. Essentially, this vulnerability could potentially lock millions of dollars in assets within the proxy indefinitely.

Any proxy using these library versions of UUPS, whose implementation contracts had not been "initialized," were at risk. Anyone could execute the initialization function, become the owner, and execute a delegatecall to a contract with a selfdestruct opcode.

### Original mitigation to attackers taking ownership of unitialized implementation contract

To mitigate this problem, the first recommendation from OpenZeppelin's engineering team was to always "initialize" implementation contracts using a constructor together with the initializer modifier, as follows.

```solidity
constructor() initializer {}
```

This was just a security measure to prevent an attacker from initializing storage variables in the implementation to become the owner. This modifier was intended to be placed in constructors in all implementation contracts of an inheritance chain.

For childmost contracts, the variable `initialSetup` will always be true. However, in parent contracts of the implementation contracts, during deployment, `initialized` will be 1 and `address(this).code.length == 0`. It is for this scenario that the `construction` variable exists—to enable the initialization of parent contracts of the implementation contract. 

In other words, the line in question is designed to account for cases such as the contract schema below, where parent contracts need to be initialized as well. The `initializer` modifier should be used; the `onlyInitializing` modifier is not intended to initialize constructors.

```solidity
contract ImplementationParent is Initializable {
    // here initialSetup will be false
    // but construction will be true
    constructor() initializer {}
}

contract ImplementationChild is Initializable, ImplementationParent {
    // here initialSetup will be true
    constructor() ImplementationParent() initializer {}
}
```

Once the implementation contract is "initialized," it is no longer possible for anyone to execute the initialization function and become the owner of the contract.

This is no longer the recommended mitigation, the recommended solution to prevent an attacker from becoming the owner of an implementation contract is shown in the next section, but the modifier `initializer` keeps the variable `construction` for backward compatibility reasons. It is possible that it will be removed in a future version of this contract.

## The _disableInitializers() function

The most recent and recommended approach proposed by OpenZeppelin to "initialize" implementation contracts is using the `__disableInitializers()` function.

The code is as follows:

```solidity
function _disableInitializers() internal virtual {
    // solhint-disable-next-line var-name-mixedcase
    InitializableStorage storage $ = _getInitializableStorage();
    if ($._initializing) {
        revert InvalidInitialization();
    }

    if ($._initialized != type(uint64).max) {
        $._initialized = type(uint64).max; // set initialized to its max value, preventing reinitializations
        emit Initialized(type(uint64).max);
    }
}
```

Therefore, currently, the recommended way to prevent vulnerabilities caused by "uninitialized" contracts is to include the following constructor in all implementation contracts:

```solidity
/// @custom:oz-upgrades-unsafe-allow constructor
constructor() {
    _disableInitializers();
}
```

The implementation contracts themselves are never upgraded, only the proxy. So setting the version to `type(uint64).max` (`_initialized` in the implementation) ensures the implementation contract will never be initialized. Locking the initializers in the implementation does not prevent the proxy from delegatecalling them, because the `_initialized` storage that would prevent the delegatecall is in the proxy.

## The _init and _init_unchained functions

The function that initializes upgradeable contracts can have any name, but OpenZeppelin contracts follow a standard. In fact, they all have two initialization functions, which uses two names: `<Contract Name>_init` and `<Contract Name>_init_unchained`.

The `<Contract Name>_init_unchained` function contains all the code that must be executed when the implementation contract is initialized. For example, in the case of an ERC20 token, this function sets the token's name and symbol.

```solidity
function __ERC20_init_unchained(string memory name_, string memory symbol_)    
    internal    
    onlyInitializing
{
    ERC20Storage storage $ = _getERC20Storage();
    $._name = name_;
    $._symbol = symbol_;
}
```

The `<Contract Name>_init` function executes `<Contract Name>_init_unchained` **as well as** `<Parent Contract>_init_unchained` for all its parents that must be initialized.

Let's consider the case of the [GovernorUpgradeable.sol v5](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/governance/GovernorUpgradeable.sol) contract, which needs to initialize itself and one of its parent contracts, the **EIP712Upgradeable** contract.

![Codesnippet of the _Governor_init functioin in GovernorUpgradeable.sol v5 contract](https://static.wixstatic.com/media/706568_6a9ccdc272414479a5a9eac6fba47ac8~mv2.png/v1/fill/w_740,h_251,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_6a9ccdc272414479a5a9eac6fba47ac8~mv2.png)

In general, it is sufficient to execute the `_init` function of a contract, which also initializes the parent contracts. Care must be taken to not initialize the same contract twice, which can occur in an inheritance chain where two contracts share the same parent.  

That is the reason why there are two functions, `_init` and `_init_unchained`. If one needs to initialize a contract without initializing its parents, the `_init_unchained` function must be used.

### Initializing an ERC20 Upgradeable contract

An example of how to initialize upgradeable contracts can be seen in the image below, showing an upgradeable ERC20 token contract generated by the OpenZeppelin Wizard.

![Upgradeable ERC20 token contract generated by the OpenZeppelin Wizard.](https://static.wixstatic.com/media/706568_485110f33f2147fa972f6d9052275f57~mv2.png/v1/fill/w_740,h_325,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_485110f33f2147fa972f6d9052275f57~mv2.png)

Note that it calls the `__<Contract Name>_init` on its parents. Regardless of the scheme used to initialize contracts, **it is essential to ensure that all contracts in the inheritance chain are properly initialized and that no contract is initialized twice or the initializers are idempotent.**

## Warnings and recommendations

Before concluding this article, some recommendations should be given for the correct use of the **Initializable.sol** contract.

1. OpenZeppelin has another contract called [Initializable.sol](https://github.com/OpenZeppelin/openzeppelin-upgrades/blob/master/packages/core/contracts/Initializable.sol) in its openzeppelin-upgrades library. This contract exists for backward compatibility reasons and should not be used in new projects. The recommended way to import it is via `import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";`.
2. Since the initialization function is a regular function there is a risk of it being frontrun by another transaction. If this happens, the proxy contract must be redeployed. To prevent this, the [ERC1967Proxy](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/ERC1967/ERC1967Proxy.sol) contract constructor makes a call to the implementation at deploy time. The initialization call must be made at this moment, encoded in the `_data` variable.  
3. As mentioned in the previous section, when the contract is part of an inheritance chain, manual care must be taken to not invoke a parent initializer twice. The schema does not identify such potential problems, so verification must be done manually. One way to solve this is to ensure all initialization functions are idempotent, meaning they have the same effect regardless of how many times they are executed.

*Originally Published Jul 8*
