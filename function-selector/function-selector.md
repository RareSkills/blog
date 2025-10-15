# Understanding the Function Selector in Solidity

The function selector is a 4-byte id that Solidity uses to identify functions under the hood.

The function selector is how a Solidity contract knows which function you are trying to call in the transaction.

You can see the 4-byte id using the `.selector` method:

```solidity!
pragma solidity 0.8.25;

contract SelectorTest{

    function foo() public {}

    function getSelectorOfFoo() external pure returns (bytes4) {
        return this.foo.selector; // 0xc2985578
    }
}
```

If a function takes no arguments, you can invoke that function by sending those 4 bytes (the selector) as data to the contract with a low level `call`.

In the following example, the `CallFoo` contract calls the foo function defined in the `FooContract` by making a call to `FooContract` with the appropriate four-byte function selector

```solidity!
pragma solidity 0.8.25;

contract CallFoo {

    function callFooLowLevel(address _contract) external {
        bytes4 fooSelector = 0xc2985578;

        (bool ok, ) = _contract.call(abi.encodePacked(fooSelector));
        require(ok, "call failed");
    }

}

contract FooContract {

    uint256 public x;

    function foo() public {
        x = 1;
    }
}
```

If `FooContract` and `CallFoo` are deployed, and the `callFooLowLevel()` function is executed with the address of `FooContract`, then the value of `x` within `FooContract` will be set to `1`. This indicates a successful call to the `foo` function.

## Identifying function calls and selector in Solidity with msg.sig

`msg.sig` is a global variable that returns the first four bytes of the transaction data, which is how the Solidity contract knows which function to invoke.

`msg.sig` is preserved across the entire transaction, it does not changed depending on which function is currently active. So even if a public function calls another public function during a call, `msg.sig` will be the selector of the function that was initially called.

In the code below, `foo` calls `bar` to get the `msg.sig`, but the selector that gets returned when `foo` is called is the function selector of `foo`, not `bar`:

You can [test the code in Remix here](https://remix.ethereum.org/#code=Ly9TUERYLUxpY2Vuc2UtSWRlbnRpZmllcjogTUlUCnByYWdtYSBzb2xpZGl0eSAwLjguMjU7Cgpjb250cmFjdCBTZWxlY3RvclRlc3QgewoKICAgIC8vIHJldHVybnMgZnVuY3Rpb24gc2VsZWN0b3Igb2YgYGJhcigpYCAweGZlYmIwZjdlCiAgICBmdW5jdGlvbiBiYXIoKSBwdWJsaWMgcHVyZSByZXR1cm5zIChieXRlczQpIHsKICAgICAgICByZXR1cm4gbXNnLnNpZzsKICAgIH0KCiAgICAvLyByZXR1cm5zIGZ1bmN0aW9uIHNlbGVjdG9yIG9mIGBmb28oKWAgMHhjMjk4NTU3OAogICAgZnVuY3Rpb24gZm9vKCkgcHVibGljIHB1cmUgcmV0dXJucyAoYnl0ZXM0KSB7CiAgICAgICAgcmV0dXJuIGJhcigpOwogICAgfQogICAgCiAgICBmdW5jdGlvbiB0ZXN0U2VsZWN0b3JzKCkgZXh0ZXJuYWwgcHVyZSByZXR1cm5zIChib29sKSB7CgogICAgICAgIGFzc2VydCh0aGlzLmZvby5zZWxlY3RvciA9PSAweGMyOTg1NTc4KTsKICAgICAgICBhc3NlcnQodGhpcy5iYXIuc2VsZWN0b3IgPT0gMHhmZWJiMGY3ZSk7CgogICAgICAgIHJldHVybiB0cnVlOwogICAgfQp9&lang=en&optimize=false&runs=200&evmVersion=null&version=soljson-v0.8.25+commit.b61c2a91.js):

```solidity~
//SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

contract SelectorTest {

    // returns function selector of `bar()` 0xfebb0f7e
    function bar() public pure returns (bytes4) {
        return msg.sig;
    }

    // returns function selector of `foo()` 0xc2985578
    function foo() public pure returns (bytes4) {
        return bar();
    }

    function testSelectors() external pure returns (bool) {

        assert(this.foo.selector == 0xc2985578);
        assert(this.bar.selector == 0xfebb0f7e);

        return true;
    }
}
```

## Solidity function signature

The function signature in Solidity is a string holding the name of the function followed by the types of the arguments it accepts. The variable names are removed from the arguments.

In the following snippet, the function is on the left, and the function signature is on the right:

```solidity!
function setPoint(uint256 x, uint256 y) --> "setPoint(uint256,uint256)"
function setName(string memory name) --> "setName(string)"
function addValue(uint v) --> "addValue(uint256)"
```

There are no spaces in the function selector. All uint types must include their size explicitly (uint256, uint40, uint8, etc). The `calldata` and `memory` types are not included. For example, `getBalanceById(uint)` is an invalid <u>signature</u>.

## How the function selector is computed from the function signature

The function selector is the first four bytes of the keccak256 hash of the function signature.

```solidtity!
function returnFunctionSelectorFromSignature(
    string calldata functionName
) public pure returns(bytes4) {
    bytes4 functionSelector = bytes4(keccak256(abi.encodePacked(functionName)));
    return(functionSelector);
}
```

Putting **“`foo()`”** into the function above will return `0xc2985578`.

The code below demonstrates computing the function selector, given the function signature.

```solidity!
//SPDX-License-Identifier: MIT
pragma solidity 0.8.25;

contract FunctionSignatureTest {

    function foo() external {}

    function point(uint256 x, uint256 y) external {}

    function setName(string memory name) external {}

    function testSignatures() external pure returns (bool) {

    // NOTE: Casting to bytes4 takes the first 4 bytes
    // and removes the rest

    assert(bytes4(keccak256("foo()")) == this.foo.selector);
        assert(bytes4(keccak256("point(uint256,uint256)")) == this.point.selector);
        assert(bytes4(keccak256("setName(string)")) == this.setName.selector);

    return true;

    }
}
```

Interfaces are encoded as addresses. For example `f(IERC20 token)` is encoded as `f(address token)`. Making an address payable does not change the signature.

## Internal functions do not have a function selector

Public and external functions have a selector but internal and private functions do not. The following code does not compile:

```solidity!
contract Foo {

    function bar() internal {
    }

    // does not compile
    function barSelector() external pure returns (bytes4) {
        return this.bar.selector;
    }
}
```

If `bar` is changed to `public` or `external`, the code will compile.

Internal functions do not need function selectors because function selectors are intended for use by external contracts. That is, the function selector is how an external user specifies which public or external function they are trying to call.

## Why use function selectors instead of the name of the function?

Solidity function names can be arbitrarily long, and if the function name is long, it will increase the size and cost of the transaction. Transactions are generally smaller in size if a function selector is provided instead of the name.

## Does the fallback function have a function selector?

The fallback function does not have a function selector.

If `msg.sig` is logged inside the fallback function, it will simply log the first four bytes of the transaction data. If there is no transaction data, then msg.sig will return four bytes of zero. This does not mean that the function selector of the fallback is all zeros, it means `msg.sig` tried to read data that wasn’t there.

```solidity!
contract FunctionSignatureTest {

    event LogSelector(bytes4);

    fallback() external payable {
        emit LogSelector(msg.sig);
    }
}
```

Here is the [**code on Remix**](https://remix.ethereum.org/#code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IE1JVApwcmFnbWEgc29saWRpdHkgMC44LjI1OwoKY29udHJhY3QgRnVuY3Rpb25TaWduYXR1cmVUZXN0IHsKCiAgICBldmVudCBMb2dTZWxlY3RvcihieXRlczQpOwoKICAgIGZhbGxiYWNrKCkgZXh0ZXJuYWwgcGF5YWJsZSB7CiAgICAgICAgZW1pdCBMb2dTZWxlY3Rvcihtc2cuc2lnKTsKICAgIH0KfQ&lang=en&optimize=false&runs=200&evmVersion=null&version=soljson-v0.8.25+commit.b61c2a91.js) to test the above contract. Below is a video showing how to trigger the fallback function on Remix:

<video src="https://video.wixstatic.com/video/706568_e6a59aac2cb04bad8146e0f232de73d7/720p/mp4/file.mp4" style="width: 100%; height: 100%;" autoplay loop muted controls></video>

## Probability of a function selector collision

A function selector can hold up to 2**32 - 1 (approximately 4.2 billion) possible values. As such, the odds of two functions having the same selector is small but realistic.

```solidity
contract WontCompile {

    function collate_propagate_storage(bytes16 x) external {}

    function burn(uint256 amount) external {}
}
```

The example above was taken from this [forum post](https://forum.openzeppelin.com/t/beware-of-the-proxy-learn-how-to-exploit-function-clashing/1070). Both of these functions have a function selector of `0x42966c68`.

## Useful resources for function selectors

[Function selector calculator](https://www.evm-function-selector.click/). Be sure to add the keyword `function`. The calculator expects a format like `function burn(uint256)`.

[Function selector database](https://www.4byte.directory/). Put the 4 bytes into the search and it will turn up the known functions that match it.

## Function selectors and the EVM — Ethereum Virtual Machine

For readers familiar with the EVM, there is a common source of confusion. The four byte function selector happens at the Solidity application level, not the EVM level. Nothing in Ethereum’s specification states that functions must be identified with 4 bytes. This is a strong convention, but it is not required.

In fact, a common [optimization for layer 2 applications](https://www.rareskills.io/post/l2-calldata) is to identify the function with a 1 byte function selector. That is, all function calls go to the fallback function, then the fallback function looks at the first byte in the transaction to determine which function to invoke.

## Learn more with RareSkills

See our [Solidity bootcamp](https://www.rareskills.io/solidity-bootcamp) for more advanced topics in smart contract development.

*Originally Published March 30, 2024*
