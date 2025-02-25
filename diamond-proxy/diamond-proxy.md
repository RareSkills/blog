# The Diamond Proxy Pattern Explained

The Diamond Pattern (ERC-2535) is a [proxy pattern](https://www.rareskills.io/proxy-patterns) where the proxy contract uses multiple implementation contracts simultaneously, unlike the [Transparent Upgradeable Proxy](https://www.rareskills.io/post/transparent-upgradeable-proxy) and [UUPS](https://www.rareskills.io/post/uups-proxy), which rely on just one implementation contract at a time. The proxy contract determines which implementation contract to [delegatecall](https://www.rareskills.io/post/delegatecall) based on the [function selector](https://www.rareskills.io/post/function-selector) of the calldata it receives (the exact mechanism is described later):

![Diamond Proxy Diagram](https://www.rareskills.io/wp-content/uploads/2025/02/DiamondDiagramMain-1.png)

One of the advantages of having multiple implementation contracts is that there is no practical upper limit on the amount of logic the proxy contract can use. Recall that the EVM limits smart contract bytecode size to 24kb. If the developer needs to deploy up to 48kb worth of bytecode, a viable solution is to use the [fallback-extension pattern](https://www.rareskills.io/post/fallback-extension-pattern). For more than 48kb, the diamond pattern is the most common solution.

In diamond pattern nomenclature, the “proxy contract” is called a diamond and the “implementation contracts” are called “facets.” Having two terms that refer to the same thing has led to confusion, so we want to drill the point home now:

- **diamond = proxy contract**
- **facet = implementation contract**

A diamond (the proxy) can be upgraded by changing one or more of the implementation contracts (facets). Alternatively, a diamond can be non-upgradeable (immutable) by not supporting a mechanism to change the facets (implementation contracts).

## Learning the Diamond Pattern

The diamond proxy has a reputation for being an “expert design pattern” because of the emergent complexity of dealing with multiple implementation contracts at the same time. In fact, the diamond pattern is somewhat controversial among EVM developers due to its alleged complexity (we do not weigh in on the debate or make a complexity judgement here).

The specification itself is fairly small. It requires four public view functions — but if the diamond is upgradeable, a fifth state-changing function for switching out the implementation contracts is required. The diamond pattern mandates only a single event (regardless of whether the contract is upgradeable). Despite the small specification, *implementing* those four (or five) functions is considerably more complex than other proxy patterns.

However, with the right prerequisites (which are considerable!), the diamond pattern is not particularly difficult to conceptualize. We assume the reader is familiar with the topics covered in the first thirteen chapters of our [Proxy Patterns Book](https://www.rareskills.io/proxy-patterns). If you haven’t read those chapters or aren’t already familiar with those topics, learning the diamond pattern will be a struggle, so make sure the prerequisites are in place.

Here are some issues that the pattern needs to deal with:

- When the diamond receives a transaction, how does it know which implementation contract to call?
- If a facet (implementation contract) is upgraded, how does the proxy contract (the diamond) know which function the new facet (implementation contract) supports, and potentially, which functions are no longer supported?
- Where should the upgrade logic be — in the proxy bytecode or in a facet?
- Since each implementation contract doesn’t directly know about the other implementations, how can storage collisions be avoided?
- How can an external actor know which functions are supported by the diamond proxy? Inheriting an interface is not sufficient since the functions could change during an upgrade.
- What if a function inside one implementation contract wants to call a function in another implementation contract — how would this transaction be facilitated?

In this article, we will show how to address all the problems above. Note that ERC-2535 does *not* require a diamond proxy to be upgradeable. The proxy can have hardcoded implementation contracts and still be a valid diamond.

To keep things simple at the start, we begin by showing the immutable diamond.

## Immutable diamond

An *immutable diamond*, also called a *static diamond* or *single cut diamond* is a proxy contract with multiple implementation contracts — and none of the implementation contracts can be upgraded. (It is possible for an upgradeable diamond to become immutable by removing the upgrade functionality, but we’ll discuss upgradeable diamonds later).

### The diamond pattern as a proxy contract

The code for the *proxy* portion of the diamond proxy should feel familiar, if compared to the [OpenZeppelin Proxy](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/proxy/Proxy.sol) used by UUPS and Transparent Upgradeable Proxy:

```solidity
// Find facet for function that is called and execute the
// function if a facet is found and return any value.
fallback() external payable {

  // get facet from function selector
  address facet = facetAddress(msg.sig);
  require(facet != address(0));
  
  // The code below is the same as OpenZeppelin Proxy.sol
  // Execute external function from facet using delegatecall and return any value.
  assembly {
    // copy function selector and any arguments
    calldatacopy(0, 0, calldatasize())
    // execute function call using the facet
    let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
    // get any return value
    returndatacopy(0, 0, returndatasize())
    // return any return value or error back to the caller
    switch result
      case 0 {revert(0, returndatasize())}
      default {return (0, returndatasize())}
  }
}
```

The only material difference from the code above to a traditional proxy are the two lines:

```solidity
address facet = facetAddress(msg.sig);
require(facet != address(0));
```

The code above gets the first four bytes of the transaction (the function selector) using `msg.sig`, then uses `facetAddress()` to determine the address of the facet that implements the function (note that `facetAddress()` could be part of the proxy logic or live inside another facet and be delegatecalled — this will be discussed later). The proxy then delegatecalls that address with the same calldata it received.

![Getting the facet address via the function selector](https://www.rareskills.io/wp-content/uploads/2025/02/GetFunctionSelector.png)

The EIP-2535 does not specify how to map function selectors to the addresses of the implementation contracts. For a static diamond, a reasonable solution is to hardcode the relationship. This will be the approach we take in the next section.

For the upgradeable diamond, hardcoding the selectors is clearly not an option, and we must rely on mappings, as we will see in the corresponding sections.

## Conditional Branching On the Function Selector

Below we show a proxy contract with two implementation contracts (facets):

- The first implementation contract `Add` exposes a single public function `add()` which returns the sum of its arguments.
- The second implementation contract `Multiply` exposes two public functions `multiply()` and `exponent()` which do what their names suggest.

The code below shows a single proxy using two implementation contracts. The `facetAddress()`  function in `Diamond` takes `msg.sig` and returns the address of the facet that implements the function with that signature, if any. The code below is not yet compliant with the diamond standard, but we will evolve the code to become compliant.

```solidity
// first implementation contract
contract Add {
		// selector: 0x771602f7
    function add(uint256 x, uint256 y) external pure returns (uint256 z) {
        z = x + y;
    }
}

// second implementation contract
contract Multiply {
		// selector: 0x165c4a16
    function multiply(uint256 x, uint256 y) external pure returns (uint256 z) {
        z = x * y;
    }
    
    // selector: 0x2f8cd8b1
    function exponent(uint256 x, uint256 y) external pure returns (uint256 z) {
		    z = x ** y;
    }
}

// proxy contract
contract Diamond  {
    address immutable ADD_ADDR;
    address immutable MULTIPLY_ADDR;
    constructor() {
        ADD_ADDR = address(new Add());
        MULTIPLY_ADDR = address(new Multiply());
    }

    function facetAddress(bytes4 selector) public view returns (address) {
        if (selector == 0x771602f7) return ADD_ADDR;
        else if (selector == 0x165c4a16) return MULTIPLY_ADDR;
        else if (selector == 0x2f8cd8b1) return MULTIPLY_ADDR;

        else return address(0);
    }

    fallback() external payable {
        address facet = facetAddress(msg.sig);
        require(facet != address(0), "Function does not exist");
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

The EIP-2535 standard does not specify the error message for calling a function that does not exist. In our case, we revert with the string `"Function does not exist"`, but a custom error would be more [gas efficient](https://www.rareskills.io/post/gas-optimization).

For simplicity, our diamond deploys the two facets in the [constructor](https://www.rareskills.io/learn-solidity/constructor), but this is not done in practice. In practice, the facets are deployed separately, and the proxy is “informed” of them later in some manner, such as through the constructor arguments or a separate function.

Some diamonds implement the facets as libraries with external functions instead of contracts, but the EIP-2535 does not require the facets to be libraries or contracts.

For simplicity, we use a series of else-if statements to match the function selector to the facet address, but this is not gas-efficient if there are a lot of options. For a static diamond, sorting the selectors in advance and then doing binary search is more efficient — but we will discuss this later.

### Corner case — Ether transfers

When ether is transferred, there will be no calldata. In this situation, `msg.sig` will return `0x00000000` and this will not map to a facet address. This is okay if the contract does not intend to receive ether. However, if the contract is expected to receive ether, or is expected to react to incoming ether, then `0x00000000` should be mapped to a function with the intended logic, or at least a function that does not revert. Keep in mind that an attacker can trigger this function both by sending no calldata or by sending `0x00000000` as calldata, so the logic needs to gracefully handle both scenarios.

To make our contract EIP-2535 compliant, we must implement four mandatory public view functions, each discussed below.

## Four Mandatory Public View Functions

### 1/4 facetAddress()

Note that `facetAddress()` is public — the EIP-2535 requires that a Diamond proxy expose a public function with the signature:

```solidity
function facetAddress(bytes4 selector) external view returns (address);
```

We are already compliant in that regard.

### 2/4 facetAddresses()

In addition to `facetAddress(bytes4 selector)`, EIP-2535 mandates a function called `facetAddresses()` (plural) which returns all the facets used by the diamond — in other words, a list of addresses. The signature is as follows:

```solidity
function facetAddresses() public view returns (address[] memory addresses);
```

Since the facets on our diamond cannot change, we simply hardcode the list of facet addresses:

```solidity
contract Diamond  {
    address immutable ADD_ADDR;
    address immutable MULTIPLY_ADDR;
    constructor() {
        ADD_ADDR = address(new Add());
        MULTIPLY_ADDR = address(new Multiply());
    }

    function facetAddress(bytes4 selector) public view returns (address) {
        if (selector == 0x771602f7) return ADD_ADDR;
        else if (selector == 0x165c4a16) return MULTIPLY_ADDR;
        else if (selector == 0x2f8cd8b1) return MULTIPLY_ADDR;

        else return address(0);
    }

	// ┌────────────────────┐
    // │                    │
    // │  THIS CODE IS NEW  │
    // │                    │
    // └────────────────────┘
    function facetAddresses() public view returns (address[2] memory addresses) {
		    addresses[0] = ADD_ADDR;
		    addresses[1] = MULTIPLY_ADDR;
    }
    // --------

    fallback() external payable {
        address facet = facetAddress(msg.sig);
        require(facet != address(0), "Function does not exist");
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

There is no enforced ordering on the list of addresses (facet addresses) returned by `facetAddresses()`.

### 3/4 facetFunctionSelectors()

Given a facet address as an argument, `facetFunctionSelectors()` returns the selectors of all the facet’s public functions. It has the following signature:

```solidity
function facetFunctionSelectors(address _facet) external view returns (bytes4[] memory facetFunctionSelectors_);
```

 We can implement it as follows:

```solidity
contract Diamond  {
    address immutable ADD_ADDR;
    address immutable MULTIPLY_ADDR;
    constructor() {
        ADD_ADDR = address(new Add());
        MULTIPLY_ADDR = address(new Multiply());
    }

    function facetAddress(bytes4 selector) public view returns (address) {
        if (selector == 0x771602f7) return ADD_ADDR;
        else if (selector == 0x165c4a16) return MULTIPLY_ADDR;
        else if (selector == 0x2f8cd8b1) return MULTIPLY_ADDR;

        else return address(0);
    }

    function facetAddresses() public view returns (address[] memory addresses) {
		    addresses[0] = ADD_ADDR;
		    addresses[1] = MULTIPLY_ADDR;
    }

	// ┌────────────────────┐
    // │                    │
    // │  THIS CODE IS NEW  │
    // │                    │
    // └────────────────────┘
    function facetFunctionSelectors(address _facet) public view returns (bytes4[] memory) {
				if (_facet == ADD_ADDR) {
						bytes4[] memory facetFunctionSelectors_ = new bytes4[](1);
						facetFunctionSelectors_[0] = 0x771602f7;
						return facetFunctionSelectors_;
				}
				else if (_facet == MULTIPLY_ADDR) {
						bytes4[] memory facetFunctionSelectors_ = new bytes4[](2);
						facetFunctionSelectors_[0] = 0x165c4a16;
						facetFunctionSelectors_[1] = 0x2f8cd8b1;
						
						return facetFunctionSelectors_;
				}
				else {
						bytes4[] memory facetFunctionSelectors_ = new bytes4[](0);
						return facetFunctionSelectors_;
				}
		}
		// --------

    fallback() external payable {
        address facet = facetAddress(msg.sig);
        require(facet != address(0), "Function does not exist");
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

### 4/4 facets()

Finally, EIP-2535 mandates a function `facets()` which takes no arguments and returns a list of structs where each struct holds a facet address and a list of that facet’s function selectors.

It has the following signature:

```solidity
struct Facet {
    address facetAddress;
    bytes4[] functionSelectors;
}

function facets() external view returns (Facet[] memory facets_);
```

The information `facets()` returns can be obtained by

- first calling `facetAddresses()` to get all the facet addresses
    - looping through each facet address, and putting that address in the field `facetAddress` of `Facet` struct
        - and calling `facetFunctionSelectors()` on the address and putting the list of function selectors in the `functionSelectors` field of the struct.

An alternative is to hardcode the answer directly in the function, which may be more efficient in some cases (for upgradeable diamonds, hardcoding obviously won’t work). In our diamond, we will implement `facets()` using the loop method, as shown below. Scroll to the bottom of this code block to see the new code:

```solidity
contract Add {
		// selector: 0x771602f7
    function add(uint256 x, uint256 y) external pure returns (uint256 z) {
        z = x + y;
    }
}

contract Multiply {
		// selector: 0x165c4a16
    function multiply(uint256 x, uint256 y) external pure returns (uint256 z) {
        z = x * y;
    }
    
    // selector: 0x2f8cd8b1
    function exponent(uint256 x, uint256 y) external pure returns (uint256 z) {
		    z = x ** y;
    }
}

contract Diamond  {
    address immutable ADD_ADDR;
    address immutable MULTIPLY_ADDR;
    constructor() {
        ADD_ADDR = address(new Add());
        MULTIPLY_ADDR = address(new Multiply());
    }

    function facetAddress(bytes4 selector) public view returns (address) {
        if (selector == 0x771602f7) return ADD_ADDR;
        else if (selector == 0x165c4a16) return MULTIPLY_ADDR;
        else if (selector == 0x2f8cd8b1) return MULTIPLY_ADDR;

        else return address(0);
    }

    function facetAddresses() public view returns (address[] memory) {
        address[] memory addresses = new address[](2);
        addresses[0] = ADD_ADDR;
        addresses[1] = MULTIPLY_ADDR;

        return addresses;
    }

    function facetFunctionSelectors(address _facet) public view returns (bytes4[] memory) {
        if (_facet == ADD_ADDR) {
            bytes4[] memory selectors = new bytes4[](1);
            selectors[0] = 0x771602f7;
            return selectors;
        }
        else if (_facet == MULTIPLY_ADDR) {
            bytes4[] memory selectors = new bytes4[](2);
            selectors[0] = 0x165c4a16;
            selectors[1] = 0x2f8cd8b1;
            return selectors;
        }
        // Return empty array for unknown facets
        return new bytes4[](0);
	  }

	// ┌────────────────────┐
    // │                    │
    // │  THIS CODE IS NEW  │
    // │                    │
    // └────────────────────┘
    struct Facet {
        address facetAddress;
        bytes4[] functionSelectors;
    }

    function facets() public view returns (Facet[] memory) {
				address[] memory fa = facetAddresses();
        Facet[] memory _facets = new Facet[](2);

        for (uint256 i = 0; i < fa.length; i++) {
            _facets[i].facetAddress = fa[i];
            _facets[i].functionSelectors = facetFunctionSelectors(fa[i]);
        }
        return _facets;
    }
    // --------

    fallback() external payable {
        address facet = facetAddress(msg.sig);
        require(facet != address(0), "Function does not exist");
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

Having implemented these four public functions, we now have implemented all the mandatory external functions of a Diamond proxy.

### IDiamondLoupe

Collectively, these four functions are defined in an interface called `IDiamondLoupe`. All diamonds must implement `IDiamondLoupe`. One can remember that a “loupe” is a small magnifying glass for looking at (”viewing”) diamonds, and each of these functions are view function

```solidity
interface IDiamondLoupe {
    struct Facet {
        address facetAddress;
        bytes4[] functionSelectors;
    }

    /// @notice Gets all facet addresses and their four byte function selectors.
    /// @return facets_ Facet
    function facets() external view returns (Facet[] memory facets_);

    /// @notice Gets all the function selectors supported by a specific facet.
    /// @param _facet The facet address.
    /// @return facetFunctionSelectors_
    function facetFunctionSelectors(address _facet) external view returns (bytes4[] memory facetFunctionSelectors_);

    /// @notice Get all the facet addresses used by a diamond.
    /// @return facetAddresses_
    function facetAddresses() external view returns (address[] memory facetAddresses_);

    /// @notice Gets the facet that supports the given selector.
    /// @dev If facet is not found return address(0).
    /// @param _functionSelector The function selector.
    /// @return facetAddress_ The facet address.
    function facetAddress(bytes4 _functionSelector) external view returns (address facetAddress_);
}
```

We are still one step from a fully compliant Diamond Proxy: logging the changes made to the diamond. This is the topic we will discuss next.

## DiamondCut — logging the facets selectors

Our running diamond example is not upgradeable. Diamonds in practice are upgradeable and can change their facets and the function selectors associated with them.

Any change in a facet must be logged — even for non-upgradeable (static) diamonds, changes (the addition of facets and functions selectors) occur at deployment and therefore need to be logged.

The theory behind logging the facet selectors is that there should be two ways to determine the function selectors supported by the diamond:
1. Using the public functions described above, or
2. Parsing the logs.

Even facets that are hardcoded like ours must be logged, so events must be emitted. In our case, the emission must occur during deployment. Such logs are required by the standard.

When we add a facet (or make any alteration such as replacing or removing), that action is called a *diamond cut*.

Diamond cut does *not* mean removing facets, as “cutting” might imply. It corresponds to altering the diamond in some way. *Any* change to a facet requires emitting a `DiamondCut` event which is defined in the upcoming code block. (One can remember this nomenclature for “cut” by noting that when an actual diamond gem is “cut” it gets an extra side — or facet — where the cut occurred).

The `DiamondCut` event is defined in a new interface named `IDiamond` . Events must be emitted every time a function is added, replaced or removed from some facet. The definition of the `DiamondCut` event is shown below:

```solidity
interface IDiamond {
    enum FacetCutAction {Add, Replace, Remove}
    // Add=0, Replace=1, Remove=2

    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }

    event DiamondCut(FacetCut[] _diamondCut, address _init, bytes _calldata);
}
```

Since `FacetCut[]` is a list, multiple facets can be changed at once.

**Unlike other proxy patterns such as [Transparent Upgradeable Proxies](https://www.rareskills.io/post/transparent-upgradeable-proxy) or [UUPS Proxies](https://www.rareskills.io/post/uups-proxy), diamonds do not have a mechanism to upgrade by changing the facet address. A facet (implementation contract) is removed when all the associated function selectors are removed. Facets are implicitly added when a function selector with a new implementation address is added.**

The `_init` and `_calldata` parameters serve the same purpose as [OpenZeppelin Initializers](https://www.rareskills.io/post/initializable-solidity) — we will discuss this more later. If no initialization data is necessary, then `_init` should be `address(0)` and `_calldata` should be `""` or empty bytes.

Let’s update our diamond to emit this event in the constructor:

```solidity
contract Add {
	// selector: 0x771602f7
    function add(uint256 x, uint256 y) external pure returns (uint256 z) {
        z = x + y;
    }
}

contract Multiply {
	// selector: 0x165c4a16
    function multiply(uint256 x, uint256 y) external pure returns (uint256 z) {
        z = x * y;
    }
    
    // selector: 0x2f8cd8b1
    function exponent(uint256 x, uint256 y) external pure returns (uint256 z) {
		    z = x ** y;
    }
}

interface IDiamond {
    enum FacetCutAction {Add, Replace, Remove}
    // Add=0, Replace=1, Remove=2

    struct FacetCut {
        address facetAddress;
        FacetCutAction action;
        bytes4[] functionSelectors;
    }

    event DiamondCut(FacetCut[] _diamondCut, address _init, bytes _calldata);
}

contract Diamond is IDiamond {
    address immutable ADD_ADDR;
    address immutable MULTIPLY_ADDR;

    constructor() {
        ADD_ADDR = address(new Add());
        MULTIPLY_ADDR = address(new Multiply());

	    // ┌────────────────────┐
	    // │                    │
	    // │  THIS CODE IS NEW  │
	    // │                    │
	    // └────────────────────┘
	
        // there are a total of 3 facets:
        // [add, [add]]
        // [multiply, [multiply, exponent]]
        // [this, [facets, facetAddress, facetAddresses, facetFunctionSelectors]]
        FacetCut[] memory _diamondCuts = new FacetCut[](3);
        enum FacetCutAction {Add, Replace, Remove}

        _diamondCuts[0].facetAddress = ADD_ADDR;
        bytes4[] memory _addFacets = new bytes4[](1);
        _addFacets[0] = 0x771602f7;
        
        // add to _diamondCuts
        _diamondCuts[0].action = Add;
        _diamondCuts[0].functionSelectors = _addFacets;

        _diamondCuts[1].facetAddress = MULTIPLY_ADDR;
        bytes4[] memory _mulFacets = new bytes4[](2);
        _mulFacets[0] = 0x165c4a16;
        _mulFacets[1] = 0x2f8cd8b1;
        
        // add to _diamondCuts
        _diamondCuts[1].action = Add;
        _diamondCuts[1].functionSelectors = _mulFacets;

        // Note that the IDiamondLoupe interface functions are also logged.
        _diamondCuts[2].facetAddress = address(this);
        bytes4[] memory _loupeFacets = new bytes4[](4);
        _loupeFacets[0] = this.facetAddress.selector;
        _loupeFacets[1] = this.facetAddresses.selector;
        _loupeFacets[2] = this.facets.selector;
        _loupeFacets[3] = this.facetFunctionSelectors.selector;

        // add to _diamondCuts
        _diamondCuts[2].action = Add;
        _diamondCuts[2].functionSelectors = _loupeFacets;

        emit DiamondCut(_diamondCuts, address(0), "");
        
        // --------
    }

    function facetAddress(bytes4 selector) public view returns (address) {
        if (selector == 0x771602f7) return ADD_ADDR;
        else if (selector == 0x165c4a16) return MULTIPLY_ADDR;
        else if (selector == 0x2f8cd8b1) return MULTIPLY_ADDR;

        else return address(0);
    }

    function facetAddresses() public view returns (address[] memory) {
        address[] memory addresses = new address[](2);
        addresses[0] = ADD_ADDR;
        addresses[1] = MULTIPLY_ADDR;

        return addresses;
    }

    function facetFunctionSelectors(address _facet) public view returns (bytes4[] memory) {
        if (_facet == ADD_ADDR) {
            bytes4[] memory selectors = new bytes4[](1);
            selectors[0] = 0x771602f7;
            return selectors;
        }
        else if (_facet == MULTIPLY_ADDR) {
            bytes4[] memory selectors = new bytes4[](2);
            selectors[0] = 0x165c4a16;
            selectors[1] = 0x2f8cd8b1;
            return selectors;
        }
        // Return empty array for unknown facets
        return new bytes4[](0);
	  }

    struct Facet {
        address facetAddress;
        bytes4[] functionSelectors;
    }

    function facets() public view returns (Facet[] memory) {
		address[] memory fa = facetAddresses();
        Facet[] memory _facets = new Facet[](2);

        for (uint256 i = 0; i < fa.length; i++) {
            _facets[i].facetAddress = fa[i];
            _facets[i].functionSelectors = facetFunctionSelectors(fa[i]);
        }
        return _facets;
    }

    fallback() external payable {
        address facet = facetAddress(msg.sig);
        require(facet != address(0), "Function does not exist");
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), facet, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }

    receive() external payable {}
}
```

We now have a fully compliant EIP-2535 Diamond proxy. Normally, the `IDiamondLoupe` functions are stored in a separate facet, but for simplicity we have put them in the diamond to keep things simple for now.

Regardless of where the functions for `IDiamondLoupe` are stored — whether in the diamond itself or another facet, the standard requires emitting their address, the action that took place (adding a facet), and function selectors.

### Duplication between IDiamondLoupe and events from DiamondCut

One of the controversial aspects of this EIP is the fact that the function selectors can be determined both by parsing the past logs and by calling the view functions in `IDiamondLoupe`. This creates duplicate logic to accomplish the same thing.

The rationale behind exposing the same data via public functions is that it makes integration with block explorers and other external systems easier. Additionally, upgrade scripts can atomically check if a function selector already exists before registering a new one. The intent behind the `DiamondCut` events is to show a history of upgrades.

In ERC-1967, a block explorer can query the storage slot and immediately identify where the logic contract is — the block explorer does not need to parse the logs emitted by ERC-1967, which contain the same information.

## Making the Diamond Upgradeable, Implementing the diamondCut Function

The EIP-2535 standard suggests adding a function `diamondCut()` shown below by which facets can be added, changed, or removed.

```solidity
interface IDiamondCut is IDiamond {
    /// @notice Add/replace/remove any number of functions and optionally execute
    ///         a function with delegatecall
    /// @param _diamondCut Contains the facet addresses and function selectors
    /// @param _init The address of the contract or facet to execute _calldata
    /// @param _calldata A function call, including function selector and arguments
    ///                  _calldata is executed with delegatecall on _init
    function diamondCut(
        FacetCut[] calldata _diamondCut,
        address _init,
        bytes calldata _calldata
    ) external;
}
```

The standard does not *require* the upgrade function to be called `diamondCut()` or to implement the signature above. A developer could use their own function `changeTheFacets()` for example — but that function must emit the `DiamondCut` event in accordance with the kind of facet update it made.

## Data structures for facets and selectors

To store the function selectors and facts, at minimum we want a mapping from function selectors to the facet address:

```solidity
mapping(bytes4 => address) facetToSelectors;
```

This data structure enables us to:

- Figure out which implementation contract to delegatecall based on the `msg.sig` received
- Determine if a new function selector can be added without colliding with a pre-existing function selector. We must require `facetToSelectors[selector] == address(0)`
- Determine if a selector can be replaced or removed. Using the same check, we can determine if we are replacing or removing a non-existent selector (this operation should revert).

## Implementing the functions for `IDiamondLoupe`

As a recap, here are the view functions in `IDiamondLoupe` we need to support:

1. `facetAddress(bytes4 selector)` given a function selector, return the facet
2. `facetAddresses()` return all the facet addresses
3. `facetFunctionSelectors(address facet)` given an address, return all the function selectors
4. `facets()` return all the facet addresses and their function selectors

Here is how we implement each one:

1. The function `facetAddress(bytes4 selector)` can simply be a view function for the `facetToSelectors` mapping — or we could make the mapping public.
2. To return all the addresses for `facetAddresses()`, we must either:
    1. Explicitly store all the addresses in a list
    2. Explicitly store all the function selectors in a list, loop through the function selectors, and call `facetAddress(bytes4 selector)` on each one, then build up a list of unique addresses.
3. To return all the function selectors of a facet via `facetFunctionSelectors(address facet)` we must either:
    1. Create a mapping `mapping(address facet => bytes4[])` that stores the list of function selectors associated with each mapping
    2. Maintain an array of all the function selectors, and loop through all that array and call `facetAddress(bytes4 selector)` on each one. Add the address to a list if the returned `facet` is the `facet` in the argument of `facetFunctionSelectors`.
4. Since `facets()` returns the same information as looping through `facetAddresses()` and calling `facetFunctionSelectors(address facet)` on each address, we omit further discussion of implementing `facets()`.

There is a fundamental tradeoff. If we use more data structures, it will be cheaper to call the view functions because they don’t have to “reconstruct” the data these functions return, but more data structures need to be updated during an upgrade. So we either must have:

- The upgrade is cheaper, but calling the view functions on-chain is more expensive.
- Calling the view function is cheaper, but upgrading a facet requires more bookkeeping (increased gas cost).

It is extremely uncommon to call any of the `IDiamondLoupe` view functions on-chain as they are intended for off-chain consumption. Therefore, opting for fewer data structures is preferred.

## Storage variables for facets and selectors — and avoiding storage collisions

As seen, to store selector and address information, we need at least a mapping such as:

```solidity
mapping(bytes4 => address) facetToAddress;
```

in the diamond, but then any mapping within a facet assigned to the first storage slot will potentially collide with this mapping.

Storage collisions in the Diamond proxy are more complicated than in other proxy patterns, because collisions can occur not only in upgrades of the same implementation contract, but also between facets.

The Diamond pattern does not specify how storage should be managed. One straightforward way to handle collisions is using storage namespaces. A detailed explanation of using namespaces can be found in [EIP-7201 storage namespaces](https://www.rareskills.io/post/erc-7201).

As a review, in storage namespaces, the state variables of a contract are grouped into a struct, and the base of this struct is stored in a pseudorandom slot, typically determined by the hash of a string. As a result, each contract has its own base storage slot, making storage collisions highly unlikely.

*EIP-7201 was derived from an earlier solution proposed by the diamond pattern called “diamond storage.” EIP-2535 also proposed another pattern called “App Storage.” However, EIP-2535 does not dictate how storage should be managed, so we simply refer the reader to a viable solution, which is to use EIP-7201. The interested reader can learn about the “diamond storage” and “app storage” patterns directly from the EIP — both of which are recommended by the EIP author.*

## The minimum storage to operate the diamond

If we chose to keep the minimum storage required to operate the diamond, the namespace for the proxy should be a struct with the following fields:

- We need a list of selectors, i.e. `bytes4[] selectors`. Every time we add a selector, we scan through this list to make sure we aren’t adding a previously existing selector.
- We need at least a mapping from selectors to addresses. However it would be helpful to also map the selectors to their index in `selectors`. That way, when we remove a selector, we can quickly look up its index in the array. Then, we swap that entry with the last entry and pop the list. So instead of storing `selector => address` we store a struct which holds the address and the selector’s position in the array. Therefore, our mapping holds `selector => (address, index_in_selectors)`.

The code below implements the above two points:

- `selectors` is simply the list of selectors
- The struct `FacetAddressAndSelectorPosition` stores the facetAddress and where the index of the selector is in `selectors`

```solidity
struct FacetAddressAndSelectorPosition {
    address facetAddress;
    uint16 selectorPosition; // index of the selector in `selectors`
}

struct DiamondStorage {
    mapping(bytes4 => FacetAddressAndSelectorPosition) facetAddressAndSelectorPosition;
    bytes4[] selectors;
}
```

The struct can be accessed using the same pattern from EIP-7201.

### LibDiamond to manage storage

To keep things simple, the struct that holds the information above and the function for setting the storage pointer to the struct can be kept in a separate library called `LibDiamond`. The library provides a function `diamondStorage()` which returns a pointer to the struct and `facetAddress(bytes4 selector)`. Defining `facetAddress` in this library is optional and purely for convenience.

```solidity

// ┌────────────────────┐
// │                    │
// │  CODE FOR STORAGE  │
// │                    │
// └────────────────────┘
library LibDiamond {
		// keccak256(abi.encode(uint256(keccak256("diamond.storage")) - 1)) & ~bytes32(uint256(0xff));
		bytes32 constant DIAMOND_STORAGE_POSITION = 0xd7ce2c87e6a71bef91a0dfa43113050aa4eae7c1a7c451ae61d9077904d7cd00;
		
    struct DiamondStorage {
        mapping(bytes4 => FacetAddressAndSelectorPosition) facetAddressAndSelectorPosition;
        bytes4[] selectors;
    }
		
		function diamondStorage()
		    internal
		    pure
		    returns (DiamondStorage storage ds) {
		    
		    bytes32 position = DIAMOND_STORAGE_POSITION;
		    assembly {
				    // change the slot of the storage pointer
		        ds.slot := position
		    }
		}
		
		function facetAddress(bytes4 _functionSelector)
		    external
		    override
		    view
		    returns (address facetAddress_) {
		    
		    LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
		    facetAddress_ = ds.facetAddressAndSelectorPosition[_functionSelector].facetAddress;
		}
}
```

For upgradeable diamonds, it is conventional to keep the `IDiamondLoupe` functions in their own facet, and not in the proxy itself. There is no requirement for what to name this contract, but `DiamondLoupeFacet` is reasonably descriptive. Below we show `DiamondLoupeFacet` using the `LibDiamond` library to implement the external `facetAddress` function that is part of `IDiamondLoupe`.

```solidity
import { LibDiamond } from "./libraries/LibDiamond.sol";
// ┌─────────────────────┐
// │                     │
// │  DiamondLoupeFacet  │
// │                     │
// └─────────────────────┘
contract DiamondLoupeFacet is IDiamondLoupe {

    /// @notice Gets the facet address that supports the given selector.
    /// @dev If facet is not found return address(0).
    /// @param _functionSelector The function selector.
    /// @return facetAddress_ The facet address.
    function facetAddress(bytes4 _functionSelector)
        external
        override
        view
        returns (address facetAddress_) {

        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        facetAddress_ = ds.facetAddressAndSelectorPosition[_functionSelector].facetAddress;
    }
    
    // other functions not shown
}
```

## Reference Implementations of the Diamond Standard

Nick Mudge, the author of EIP-2535 maintains three reference implementations (diamond-1, diamond-2, and diamond-3) in the following repo:

[https://github.com/mudgen/diamond](https://github.com/mudgen/diamond)

These implementations optimize for the tradeoffs we discussed earlier: if the view functions in `IDiamondLoupe` are cheap to query on-chain, they will be expensive to update and vice versa.

Diamond-1 and diamond-2 use as little storage as possible to track the facets and selectors, only using a list of function selectors and a mapping from the function selector to the facet address. Below we see the storage for diamond-1.

Note that the reference implementation stores implements the array of function selectors as a mapping from `uint256 => bytes32` selector slots to pack 8 function selectors into one slot. Mappings are slightly more gas efficient than arrays because they do not implicitly check the length of the array before doing a lookup. The length of this “array” is stored separately as `selectorCount`.

```solidity
struct DiamondStorage {
    // function selector => facet address and selector position in selectors array
    mapping(bytes4 => FacetAddressAndSelectorPosition) facetAddressAndSelectorPosition;
    bytes4[] selectors;
    mapping(bytes4 => bool) supportedInterfaces;
    // owner of the contract
    address contractOwner;
}
```

Diamond-3 on the other hand explicitly stores the facet addresses and a mapping from the facet address to the list of function selectors it stores:

```solidity
struct FacetAddressAndPosition {
    address facetAddress;
    uint96 functionSelectorPosition; // position in facetFunctionSelectors.functionSelectors array
}

struct FacetFunctionSelectors {
    bytes4[] functionSelectors;
    uint256 facetAddressPosition; // position of facetAddress in facetAddresses array
}

struct DiamondStorage {
    // maps function selector to the facet address and
    // the position of the selector in the facetFunctionSelectors.selectors array
    mapping(bytes4 => FacetAddressAndPosition) selectorToFacetAndPosition;
    // maps facet addresses to function selectors
    mapping(address => FacetFunctionSelectors) facetFunctionSelectors;
    // facet addresses
    address[] facetAddresses;
    // Used to query if a contract implements an interface.
    // Used to implement ERC-165.
    mapping(bytes4 => bool) supportedInterfaces;
    // owner of the contract
    address contractOwner;
}
```

The DiamondLoupe implementation for diamond-3 is very simple as it is simply a thin wrapper over those storage variables:

```solidity
contract DiamondLoupeFacet is IDiamondLoupe, IERC165 {
    // Diamond Loupe Functions
    ////////////////////////////////////////////////////////////////////
    /// These functions are expected to be called frequently by tools.
    //
    // struct Facet {
    //     address facetAddress;
    //     bytes4[] functionSelectors;
    // }

    /// @notice Gets all facets and their selectors.
    /// @return facets_ Facet
    function facets() external override view returns (Facet[] memory facets_) {
        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        uint256 numFacets = ds.facetAddresses.length;
        facets_ = new Facet[](numFacets);
        for (uint256 i; i < numFacets; i++) {
            address facetAddress_ = ds.facetAddresses[i];
            facets_[i].facetAddress = facetAddress_;
            facets_[i].functionSelectors = ds.facetFunctionSelectors[facetAddress_].functionSelectors;
        }
    }

    /// @notice Gets all the function selectors provided by a facet.
    /// @param _facet The facet address.
    /// @return facetFunctionSelectors_
    function facetFunctionSelectors(address _facet) external override view returns (bytes4[] memory facetFunctionSelectors_) {
        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        facetFunctionSelectors_ = ds.facetFunctionSelectors[_facet].functionSelectors;
    }

    /// @notice Get all the facet addresses used by a diamond.
    /// @return facetAddresses_
    function facetAddresses() external override view returns (address[] memory facetAddresses_) {
        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        facetAddresses_ = ds.facetAddresses;
    }

    /// @notice Gets the facet that supports the given selector.
    /// @dev If facet is not found return address(0).
    /// @param _functionSelector The function selector.
    /// @return facetAddress_ The facet address.
    function facetAddress(bytes4 _functionSelector) external override view returns (address facetAddress_) {
        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        facetAddress_ = ds.selectorToFacetAndPosition[_functionSelector].facetAddress;
    }

    // This implements ERC-165.
    function supportsInterface(bytes4 _interfaceId) external override view returns (bool) {
        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        return ds.supportedInterfaces[_interfaceId];
    }
}
```

The view functions for [diamond-1](https://github.com/mudgen/diamond-1-hardhat/blob/main/contracts/facets/DiamondLoupeFacet.sol) and [diamond-2](github.com/mudgen/diamond-2-hardhat/blob/main/contracts/facets/DiamondLoupeFacet.sol) are more complex because they have to loop through all the function selectors to list out the facet addresses, then only return the unique addresses.

## Deploying a diamond

The process of deploying and upgrading a diamond is the same as other upgradeable proxy patterns:

During deployment we:

1. Deploy the facets (implementations)
2. Deploy the proxy with the list of facets and selectors as the arguments to the constructor, and initialize in the same transaction

And during an upgrade:

1. Deploy the new facet(s)
2. Call diamondCut() or the user-defined function with the list of selectors to remove and the list of selectors to add. It is possible to add several new facets in a single transaction.

## Initializing storage variables during deployment and upgrade

When deploying a proxy, we might want to set some initial state variables as if we had a constructor. Similarly, we might want to initialize some storage variables after an upgrade, similar to `upgradeToAndCall` in the OpenZeppelin implementation of Transparent Upgradeable Proxy and UUPS.

This is the use for the last two arguments `_init` and `_calldata` in `diamondCut`:

```solidity
function diamondCut(Facets[] facets, address _init, bytes memory _calldata)
```

If `_init ≠ address(0)` then `diamondCut` must delegatecall to `_init` using `_calldata` as the argument. Since `diamondCut` runs in the context of the proxy (diamond), the delegatecalled contract (`_init`) can initialize the storage variables in the proxy.

The initialization logic is run by an external contract. If we want to set multiple storage variables in a single transaction, it’s easier to do this as a single transaction by using a special purpose smart contract.

An example contract is shown below:

```solidity
import {LibDiamond} from "../libraries/LibDiamond.sol";

contract DiamondInit {    

    function init() external {
		    // read the ds struct from storage
		    // (remember, this executes in the context of the proxy)
        LibDiamond.DiamondStorage storage ds = LibDiamond.diamondStorage();
        
        // write to it
        ds.owner = 0x456....;
        ds.usdc = 0xa1...;
				// ...
    }
}
```

The call to `diamondCut` would pass the address of `DiamondInit` and the ABI encoding of `init()`.

Initialization can happen whenever a `diamondCut` transaction happens — it is not limited to the initial deployment of the facets. We can also delegatecall an external contract atomically when replacing or removing a function selector.

### Consider requiring a magic value with DiamondInit

EIP-2535 does not require a safety check that the DiamondInit contract is actually a contract, but it is certainly safer to check that the address for `_init` actually has bytecode. For added safety, [ZkSync’s diamond implementation](https://github.com/code-423n4/2022-10-zksync/blob/main/ethereum/contracts/zksync/libraries/Diamond.sol#L285-L288) checks that the delegatecall to the `_init` contract returns a magic value.

## Implementation details for the diamond standard

The safest way to use the diamond standard is to use the reference contracts linked above, as these have been audited. If you do not intend to call the `IDiamondLoupe` functions on-chain (which is usually the case), then diamond-2 will be the most gas efficient as it uses the least number of storage variables necessary to be compliant with ERC-2535.

### Non-upgradeable diamonds — use binary search instead of a mapping

For non-upgradeable diamonds, we recommend hardcoding the relationship between the function selectors. Instead of using a long if-else statement to find the facet address, use binary search.

As an example, consider the following function which takes a function selector as an argument and returns an address (the code is taken from [Pendle Finance](https://gist.github.com/mudgen/7d50e3d0aca93b9cc321dfa0dbded373), which uses a Diamond Proxy):

![Pendle Finance facetAddress() code](https://www.rareskills.io/wp-content/uploads/2025/02/PendleBinarySearch.png)

The code does a binary search on the function selector and returns the address of the implementation contract (facet) that holds the function. That’s all.

The addresses above are immutable addresses set in the constructor, which also emits the DiamondCut event as required:

![PendleRouter constructor](https://www.rareskills.io/wp-content/uploads/2025/02/Pendlerouter.png)

### Do not use libraries that write to storage variables unless they use EIP-7201

Any facet that writes or reads from a non-namespaced storage is liable to have a storage collision. We recommend using EIP-7201 to manage all the storage.

### Calling functions in another facet

When a function in a facet (implementation contract) runs, it does so in the context of the diamond (the proxy contract). Therefore, to call a public function in another facet, we can call the proxy address.

Consider our initial example where we had a facet `Add` with a function `add()` and a separate facet `Multiply`. Suppose we wanted to call `add()` from the `Multiply` facet. Below we show how to accomplish that; the `Multiply` facet calls the `add()` function by specifying the address of the diamond proxy:

```solidity
interface IAdd {
	function add(uint256 x, uint256 y) external view returns (uint256);
}

contract Multiply {

	function callAdd(uint256 x, uint256 y) external {
			uint256 sum = IAdd(address(this)).add(x, y);
			// rest of the code 
	}
}
```

This will do two calls under the hood:

1. First `callAdd` calls itself (in the context of the proxy)
2. The proxy matches the function selector to the Add facet
3. The proxy delegatecalls the Add facet

It may come as a surprise to some developers that a contract can call themselves in this manner, but this is indeed allowed by the EVM.

You can test the following contract to demonstrate this:

```solidity
contract SelfCall {

    uint256 public x = 0;

    function setToOne() external {
        x = 1;
    }

    function selfCall() external {
        SelfCall(address(this)).setToOne();

        // alternatively,
        // address(this).call(abi.encodeWithSignature("setToOne()"));
    }
} 
```

However, doing a self-call is a bit wasteful. To save gas, one facet can “directly” delegatecall the other facet. Behind the scenes, the calling facet’s logic runs in the proxy, and the logic is making a delegatecall to another facet, so that facet can “see” the storage variables in the proxy. The code to accomplish this will not be as clean because delegatecalls can only be a low-level call. Here is an example taken from EIP-2535:

```solidity
// get the mapping from selector => facet address
DiamondStorage storage ds = diamondStorage(); // EIP-7201

// compute the selector
bytes4 functionSelector = bytes4(keccak256("functionToCall(uint256)"));

// get the facet address
address facet = ds.selectorToFacet[functionSelector];

// delegatecall
bytes memory myFunctionCall = abi.encodeWithSelector(functionSelector, 4);
(bool success, bytes memory result) = address(facet).delegatecall(myFunctionCall);
```

However, the code above isn’t elegant and requires and additional three to four lines to do a simple call.

A third solution is the create a Solidity library(ies) that contain only internal functions, then import  those internal functions within any facet that needs them. This may lead to duplicate bytecode between facets, but if the facets don’t exceed the 24kb limit or excessively increase the deployment cost, this shouldn’t be a major issue.

## Example Diamond

We will build a diamond that serves as a counter contract. One facet will hold the view function of the counter’s value and another facet will hold the logic to increment the counter.

To illustrate the diamond init logic, we will have our counter start at 8 instead of 0.

To get started, fork the [diamond-2 reference hardhat repository](https://github.com/mudgen/diamond-2-hardhat/). We are not aware of any audited Foundry implementations.

### IncrementLibrary

It’s helpful to put the code for reading the namespace storage in a library the facets can import. Putting the logic for updating the counter inside the library is a design choice. Putting the increment logic directly in the library will increase the bytecode size for every other facet that calls the functions in that library. However, it also simplifies the logic for the increment facet, as the increment facet does not need to know anything about how the namespaced storage is structured.

Note that all the functions must be internal in this library, as the Solidity compiler expects libraries with external functions to be deployed separately.

```solidity
pragma solidity ^0.8.0;

library LibInc {
		// keccak256(abi.encode(uint256(keccak256("RareSkills.Facet.Increment")) - 1)) ^ bytes32(uint256(0xff))
		bytes32 constant STORAGE_LOCATION = 0xfa04c3581a2244f8cd60ed05a316a89d13b0e00f0bfbe2b8a2155985a9d65e00;
		
		struct IncrementStorage {
				uint256 x;
		}
		
		function incStorage()
				internal
				pure
				returns (IncrementStorage storage iStor) {
				
				bytes32 location = STORAGE_LOCATION;
        assembly {
            iStor.slot := location
        }
		}

    function x()
		    internal
		    view
		    returns (uint256 x) {
		    
        x = incStorage().x;
    }

    function increment() internal {
        incStorage().x++;
    }
}

```

Add the file `contracts/libraries/LibInc.sol`

### Initializing the storage variable with DiamondInit

Initialization happens in a separate contract that the `diamondCut()` contract delegatecalls. In the reference implementation, this is in [contracts/upgradeInitializers/DiamondInit.sol](https://github.com/mudgen/diamond-2-hardhat/blob/main/contracts/upgradeInitializers/DiamondInit.sol).

We do not show the entire contract for brevity. Add the following code to `DiamondInit.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

// ADD THIS LINE
import {LibInc} from "../libraries/LibInc.sol";
//... 

contract DiamondInit {    

    // You can add arguments to this function in order to pass in 
    // data to set your own state variables
    function init() external {
        // ...

				// ADD THESE LINES
        LibInc.IncrementStorage storage _is = LibInc.incStorage();
        _is.x = 8; // initialize x to 8
    }
}
```

### Facet that only views x

Create a new file `LibIncFacet.sol` in `contracts/facets/`:

```solidity
pragma solidity ^0.8.0;

import { LibInc } from "../libraries/LibInc.sol";

contract IncViewFacet {

   function x() external view returns (uint256 x) {
   		  x = LibInc.x();
   }
}

```

### Facet that only increments x

Create `IncrementFacet.sol` in `contracts/facets`:

```solidity
import { LibInc } from "../libraries/LibInc.sol";

contract IncrementFacet {
		function increment() external {
        LibInc.increment();
		}
}
```

### Add the following test

Add the following test to `test/diamondTest.js`

```solidity
  it.only('test increment', async () => {
    const incViewInterface = await ethers.getContractAt('IncViewFacet', diamondAddress)

    const initialX = await incViewInterface.x()
    console.log(initialX.toString())
    assert.equal(initialX, 8)

    const incrementInterface = await ethers.getContractAt('IncrementFacet', diamondAddress)
    await incrementInterface.increment();

    const afterX = await incViewInterface.x()
    assert.equal(afterX, 9)
  });
```

### Update scripts/deploy.js to deploy the new facets

Open `scripts/deploy.js` and add the contract names of the facets. Hardhat is smart enough to find the contracts without specifying the file paths. The change is shown in the red box below:

![Highlighted code changes](https://www.rareskills.io/wp-content/uploads/2025/02/Deployjs.png)

### Update the test

Update the `before` hook in the `diamondTest.js` file as follows to deploy the new facets.

Quick aside: the Solidity compiler is able to output the function selectors of public/external functions automatically. For example, suppose we have the following contract stored in `C.sol`:

```solidity
contract C {
		function foo() public {}
		function bar() external {}
}
```

If we run `solc --hashes C.sol` on the contract, we will get the following output:

```solidity
======= C.sol:C =======
Function signatures:
febb0f7e: bar()
c2985578: foo()
```

The scripts provided in this repository extract the function selectors for us using this technique, which saves us the trouble of explicitly specifying the function selectors ourselves.

Again, hardhat is able to find the contracts without specifying the file paths:

![Hardhat diff screenshot](https://www.rareskills.io/wp-content/uploads/2025/02/TerminalOutput.png)

Run the test with

```bash
npx hardhat test
```

Note that the other tests will fail because they aren’t expecting the new selectors. We ignore this by using the `.only` modifier on our test. The `.only` modifier prevents the other unit tests from being run and runs only the unit test modified with `.only`.

If you run into issues, be sure you are using Node version 20.

## Summary

- The diamond pattern is a proxy that has multiple implementation contracts.
- The diamond knows which facet to delegatecall based on the function selector of the incoming calldata.
- If the diamond is upgradeable, the mapping from selector to implementation address can be changed via diamondCut. The logic to change this mapping is not dictated by the standard, it can be part of the proxy bytecode or part of a facet.
- All facets and selectors in the diamond must be determinable via the events emitted and the public functions in IDiamondLoupe.
- Gas costs for viewing functions in `IDiamondLoupe` can be reduced by adding more data structures. However, this increases the upgrade cost. In nearly all cases, we should opt for fewer data structures because `IDiamondLoupe` functions are intended for offchain users.

*We would like to thank [Nick Mudge](perfectabstractions.com) for providing comments on an earlier version of this article.*
