# Try Catch and all the ways Solidity can revert

This article describes all the kinds of errors that can happen when a smart contract is called, and how the Solidity Try / Catch block responds (or fails to respond) to each of them.

To understand how Try / Catch works in Solidity, we must understand what data is returned when a [low-level call](https://www.rareskills.io/post/low-level-call-solidity) fails. The compiler dictates this behavior, not the Ethereum Virtual Machine (EVM). Therefore, contracts written in alternative languages or assembly will not necessarily adhere to all the error formatting explained here.

When a low-level call to an external contract fails, it returns a boolean value `false`. This `false` indicates that the call did not execute successfully. The call can return `false` in the following cases:

-   The called contract reverts
-   The called contract does an illegal operation (like dividing by zero or accessing an out-of-bounds array index)
-   The called contract uses up all the gas

[Self-destructing](https://docs.soliditylang.org/en/latest/introduction-to-smart-contracts.html#deactivate-and-self-destruct) a contract does not cause the low-level call to return `false` on EVM-compatible chains that allow self-destructing deployed contracts.

In the following sections, we will examine 10 scenarios that could cause a low-level call to return false, along with any return data they might provide.

Then we’ll explore how Try / Catch handles (or fails to handle) each situation.

## Part 1: What gets returned during a revert

## 1. What gets returned from a `revert` with no error string?

The simplest way to use `revert` is without providing a reason for the revert.

```solidity!
contract ContractA {
    function mint() external pure {
        revert();
    }
}
```

If we deploy the above contract (`ContractA`) and perform a low-level call to the `mint()` function from another contract (`ContractB`) like so:

```solidity!
// SPDX-License-Identifier: GPL-3.0
pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";

contract ContractB {
    function call_failure(address contractAAddress) external {
        (, bytes memory err) = contractAAddress.call(
            abi.encodeWithSignature("mint()")
        );

        console.logBytes(err);
    }
}
```

The `revert()` error will be triggered and no data will be returned, as shown in the screenshot below:

![revert with no error string return data in hexadecimal format: 0x](https://static.wixstatic.com/media/706568_56c8fed0267e4f0caa75b2f6a3b1a127~mv2.png/v1/fill/w_666,h_400,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_56c8fed0267e4f0caa75b2f6a3b1a127~mv2.png)

In the image above, we can see that the return data from the error is `0x` which is just a hexadecimal notation without accompanying data.

## 2. What gets returned from a `revert` with an error string?

Another way to use `revert` is by providing a string message. This helps identify why a transaction failed in your contract.

Let’s trigger a revert with a string following our previous example and see what gets returned:

```solidity!
contract ContractB {

    function mint() external pure {
        revert("Unauthorized");
    }

}
```

And the calling contract will be:

```solidity!
import "hardhat/console.sol";

contract ContractA {
    function call_failure(address contractBAddress) external {
        (, bytes memory err) = contractBAddress.call(
            abi.encodeWithSignature("mint()")
        );

        console.logBytes(err); // just so we can see the error data
    }
}
```

If we deploy both contracts and execute `ContractA` with the contract address of `ContractB` we should get the result below:

![revert with a string argument returns the ABI encoding of the Error function Error(string) ](https://static.wixstatic.com/media/706568_84eb12d0f8f14f229a5a77b3c56660b1~mv2.png/v1/fill/w_666,h_357,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_84eb12d0f8f14f229a5a77b3c56660b1~mv2.png)

When `revert` is triggered with a string argument, it returns the ABI encoding of the Error function `Error(string)` to the caller.

The returned data for our revert will be the [ABI encoding of the function call](https://www.rareskills.io/post/abi-encoding) `Error("Unauthorized")`.

In this case, it will have the [function selector](https://www.rareskills.io/post/function-selector) of the `Error(string)` function, the offset of the string, the length, and the content of the string encoded in hexadecimal.

![ABI encoding of Error("Unauthorized")](https://static.wixstatic.com/media/706568_1aa0feb1af324c4c83508ae2e77e210e~mv2.png/v1/fill/w_666,h_86,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_1aa0feb1af324c4c83508ae2e77e210e~mv2.png)

Let’s explain the output further:

-   The selector `08c379a0` is the first four bytes of `keccak256("Error(string)")` where string refers to the reason string. The next 96 bytes (3 lines) are the [ABI encoding](https://www.rareskills.io/post/abi-encoding) of the string `Unauthorized`
-   The first 32 bytes are the offset to the location of the length of the string.
-   The second 32 bytes are the length of the string (12 bytes represented in hex as `c`)
-   The actual content of the string `Unauthorized` is UTF-8 encoded as the bytes `556e617574686f72697a6564`

## 3. What gets returned from a custom revert?

[Solidity 0.8.4](https://github.com/ethereum/solidity/releases/tag/v0.8.4) introduced the error type which can be used with the revert statement to create custom errors that are both readable and gas-efficient.

To create a custom error type you’ll use the keyword error to define an error, similar to how you define [events](https://www.rareskills.io/post/ethereum-events):

```solidity
error Unauthorized();
```

You can also define custom errors with arguments if you need to emphasize some details as part of the error information: `error CustomError(arg1, arg2, etc)`.

```solidity
error Unauthorized(address caller);
```

### Custom revert without arguments

Let's compare an example of a custom revert with an argument to one without:

```solidity!
pragma solidity >=0.8.4;

error Unauthorized();

contract ContractA {

    function mint() external pure {
        revert Unauthorized();
    }

}
```

In the above example, we want to revert the transaction and return the error `Unauthorized`. Our calling contract will remain the same:

```solidity!
import "hardhat/console.sol";

contract ContractB {

    function call_failure(address contractAAddress) external {
        (, bytes memory err) = contractAAddress.call(
            abi.encodeWithSignature("mint()")
        );

        console.logBytes(err); // just so we can see the error data
    }
}
```

The custom revert (without arguments) will return only the function selector (the first four bytes of `keccak256("Unauthorized()"))` of the Unauthorized error to the caller, which is `0x82b42900`.

![Custom revert without arguments return the function selector, 0x82b42900](https://static.wixstatic.com/media/706568_7731b0a5394e44b5a88aed633df3117e~mv2.png/v1/fill/w_666,h_406,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_7731b0a5394e44b5a88aed633df3117e~mv2.png)

### Custom revert with arguments

If your custom `revert` has arguments, it’ll return the ABI encoding of the custom error function call. Here is an example:

```solidity!
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.8.4;

error Unauthorized(address caller);

contract ContractA {

    function mint() external view {
        revert Unauthorized(msg.sender);
    }

}
```

The calling contract will remain the same and the result of the error will look like this:

![Custom revert with arguments return value](https://static.wixstatic.com/media/706568_6cefb1c2997b4e57979fc08182724994~mv2.png/v1/fill/w_666,h_374,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_6cefb1c2997b4e57979fc08182724994~mv2.png)

It simply contains the ABI encoding of the custom error `Unauthorized(address)`. The encoding includes the function selector and the address argument. Since the address is a static type, its encoding is straightforward.

Here’s the structure:

-   The first four bytes represent the function Selector: `0x8e4a23d6`
-   The next 32 bytes represents the address of the caller: `0000000000000000000000009c84abe0d64a1a27fc82821f88adae290eab5e07`

As a side note, you can’t define custom `error Error(string)` or `error Panic(uint256)` since those conflict with the errors that require and assert returns respectively (we will get to `assert` in a later section).

## 4. What gets returned from revert due to require statement?

The require statement is another way to trigger a revert without using an if statement. For instance, instead of writing:

```solidity!
if (msg.sender != owner) {
   revert();
}
```

You can use the require statement like this:

```solidity
require(msg.sender == owner);
```

When `require(false)` is called without an error message, it reverts the transaction with no data, similar to `revert()`. The resulting output is an empty data payload (`0x`).

![require statement without error message revert return value](https://static.wixstatic.com/media/706568_ec1dbf70e9d143ca8583efc3299b8278~mv2.png/v1/fill/w_666,h_364,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_ec1dbf70e9d143ca8583efc3299b8278~mv2.png)

Similar to `revert` with a string, when `require` with a string like `require(false, "Unauthorized")` is triggered, it returns the ABI encoding of the error function `Error(string)`.

## 5. What gets returned from `require(false, CustomError())`?

Since May 21, 2024, custom errors have been released for the `require` statement; however, they can currently only be used through [via-ir](https://docs.soliditylang.org/en/v0.8.26/ir-breaking-changes.html). (see this [video from the Solidity team describing `via-ir`](https://www.youtube.com/watch?v=3ljewa1__UM&list=PLX8x7Zj6VeznJuVkZtRyKwseJdrr4mNsE)).

> [`via-ir`](https://docs.soliditylang.org/en/v0.8.26/ir-breaking-changes.html) [in Solidity](https://docs.soliditylang.org/en/v0.8.26/ir-breaking-changes.html) is a compilation pipeline that uses an intermediate representation (IR) in Yul to optimize your Solidity code. It’s not enabled by default, so you’ll need to use the `--via-ir` flag with [`solc`](https://github.com/ethereum/solc-js) or configure it in your preferred development environment.

### Enabling via-ir in Foundry

If you are using [Foundry](https://www.rareskills.io/post/foundry-testing-solidity), you just need to set `via-ir` to `true` in the `foundry.toml` config file to activate it like so:

```solidity!
[profile.default]
…
via-ir = true
```

### Enabling via-ir in Hardhat

In HardHat, add `viaIR:true` to your `hardhat.config` file like so:

```solidity
module.exports = {
  solidity: {
    settings: {
      viaIR: true,
    },
  },
};
```

### Enabling via-ir in Remix

If you use [Remix](https://remix.ethereum.org/), you’ll need to enable the configuration file in Advance Compiler Configurations settings, as shown in the screenshot below:

![how to enabe via-ir in remix](https://static.wixstatic.com/media/706568_e83b2eccc0cd40fc80e58a3b8efa2f48~mv2.png/v1/fill/w_666,h_891,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_e83b2eccc0cd40fc80e58a3b8efa2f48~mv2.png)

Create an empty `compiler_config.json` file in the root directory. And add the path in the configuration as shown in the above image.

Once the “Use configuration file” option is enabled, update the configuration file to include `"viaIR":true` in the settings as shown below. You might get some lint errors but your code will compile successfully.

![setting the "viaIR":true in the configuration json file](https://static.wixstatic.com/media/706568_961a1e1ad2254a6a9b8b833b40353290~mv2.png/v1/fill/w_666,h_359,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_961a1e1ad2254a6a9b8b833b40353290~mv2.png)

Once that is done, you can write your custom error with `require` like this:

```solidity
require(msg.sender == owner, Unauthorized());
```

Which is the same as this:

```solidity
if (msg.sender != owner) {
    revert Unauthorized();
}
```

And yes, it returns the same output as the custom revert we already discussed.

## 6. What gets returned from an assert?

When an `assert` statement fails, it triggers a `Panic(uint256)` error. The return value is the concatenation of the function selector (the first 4 bytes of `keccak256(”Panic(uint256)”))` and the error code.

The following code will be used to illustrate this, note the `assert` in `contractB`:

```solidity!
import "hardhat/console.sol";

contract ContractB {
    function mint() external pure {
        assert(false); // we will test what this returns
    }

}

contract ContractA {
    function call_failure(address contractBAddress) external {
        (, bytes memory err) = contractBAddress.call(
            abi.encodeWithSignature("mint()")
        );

        console.logBytes(err);
    }
}
```

When we deploy and execute the contract, we’ll get the assert error as shown below:

![assert error return value](https://static.wixstatic.com/media/706568_92337ecd3f6b457f98a17def31a55d64~mv2.png/v1/fill/w_666,h_361,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_92337ecd3f6b457f98a17def31a55d64~mv2.png)

The `err` will hold the following data:

```solidity!
0x4e487b71 // <- the function selector
0000000000000000000000000000000000000000000000000000000000000001 // the error code
```

`4e487b71` is the first four bytes of `keccak256("Panic(uint256)")` where the `uint256` refers to the error code. In this case, the error code is `1`. We’ll see other error codes in the following section.

## 7. What gets returned from an illegal operation?

Just like the `assert` statement, when illegal operations such as division by zero, popping an empty array, or an array-out-of-bounds error occur, the transaction panics and returns a concatenation of the function selector—the first 4 bytes of `keccak256("Panic(uint256)")`, and the `uint256` error code.

Here is an example of an illegal operation; array-out-of-bound — the `outOfbounds()` function in `ContractB` below has just 3 elements in the `numbers` array.

```solidity!
import "hardhat/console.sol";

contract ContractB {
    uint256[] numbers;

    constructor() {
        numbers.push(1);
        numbers.push(2);
        numbers.push(3);
    }

    function outOfbounds(uint256 index) public view returns (uint256) {
        return numbers[index];
    }
}

contract ContractA {

    function call_failure(address contractBAddress) external {
        (, bytes memory err) = contractBAddress.call(
            abi.encodeWithSignature("outOfbounds(uint256)", 10)
        );

        console.logBytes(err);
    }
}
```

If we try to access the 10th item —which of course, does not exist, we’ll get the array-out-of-bound error:

![Illegal array access operation error, out-of-bound error: 0x32](https://static.wixstatic.com/media/706568_b2d188e1f9534231a000f01f3a89e78d~mv2.png/v1/fill/w_666,h_391,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_b2d188e1f9534231a000f01f3a89e78d~mv2.png)

The `err` will hold:

```solidity!
0x4e487b71 //<- function selector for Panic(uint256)
0000000000000000000000000000000000000000000000000000000000000032 // <-the error code
```

`0x32` is the error code for out-of-bounds array error.

Here’s another example, what if we try to divide by zero?

```solidity!
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract ContractB {

    function divide(uint256 a, uint256 b) public pure returns (uint256) {
        return a / b;
    }

}
```

…and then calling the function `divide` with parameters `10` and `0`:

```solidity!
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";

contract ContractA {

    function call_failure(address contractBAddress) external {

        (, bytes memory err) = contractBAddress.call(
            abi.encodeWithSignature("divide(uint256, uint256)", 10, 0)
        );

        console.logBytes(err);
    }
}
```

The result will be the same function selector, and the error code for division by zero which is `0x12`.

![illegal division by zero error: 0x12](https://static.wixstatic.com/media/706568_8df764b9895948468411f27c7ff2aa37~mv2.png/v1/fill/w_666,h_393,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_8df764b9895948468411f27c7ff2aa37~mv2.png)

## 8. What does division by zero at the Solidity level vs. at the assembly level return?

Division by zero at the Solidity level triggers a revert with an error code of 18 (0x12). However, division by zero at the assembly level doesn’t revert, instead it returns 0. That’s because the compiler inserts checks at the Solidity level, which is not done at the assembly level.

![Division by zero error return value in solidity (18) vs assembly (0)](https://static.wixstatic.com/media/706568_6a4ec6691295405da74b257c2da785ff~mv2.png/v1/fill/w_666,h_372,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_6a4ec6691295405da74b257c2da785ff~mv2.png)

If you're performing a division operation at the assembly level, make sure to check the denominator. If it's zero, trigger a revert to roll back the transaction.

```solidity!
function divideByZeroInAssembly(uint256 numerator, uint256 denominator)
        public
        pure
        returns (uint256 result)
{
    assembly {
        if iszero(denominator) {
            revert(0, 0)
    	}
        
		result := div(numerator, denominator)
   }
}
```

But this error won’t be handled like a regular division by zero from solidity that panics with an error code of 18 in decimal or 0x12 in hex.

If you use OpenZeppelin, you can leverage the OZ [custom Panic utility](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Panic.sol) to trigger a Panic with your custom error code like so:

![how to use OZ's custom Panic Library with error code 0x12 (18 in decimals)](https://static.wixstatic.com/media/706568_26d3329115de4742890e9713ad9b3dea~mv2.png/v1/fill/w_666,h_283,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_26d3329115de4742890e9713ad9b3dea~mv2.png)

With this, you can simulate the normal behavior of `assert` at the Solidity level.

### Error Codes

According to the [Solidity docs, the following error codes](https://docs.soliditylang.org/en/latest/control-structures.html#panic-via-assert-and-error-via-require) refer to the different kinds of panics that can happen as the screenshot below shows:

![List of error codes referring to the different kinds of panics in Solidity](https://static.wixstatic.com/media/706568_38d7fd76409a4abf9a30f2f92ce930c8~mv2.png/v1/fill/w_666,h_280,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_38d7fd76409a4abf9a30f2f92ce930c8~mv2.png)

## 9. What gets returned during an out-of-gas?

During an out-of-gas error situation in a low-level call, nothing gets returned to the calling contract. No data, no error messages.

```solidity!
contract E {

    function outOfGas() external pure {
        while (true) {}
    }

}

contract C {

    function call_outOfGas(address e) external returns (bytes memory err) {
        (, err) = e.call{gas: 2300}(abi.encodeWithSignature("outOfGas()"));
    }

}
```

The variable `err` will be empty. Due to the [63/64 rule for ga](https://www.rareskills.io/post/eip-150-and-the-63-64-rule-for-gas)[s](https://www.rareskills.io/post/eip-150-and-the-63-64-rule-for-gas), contract `C` will still have 1/64th of the original gas left, so the transaction to the function `call_outOfGas` itself will not necessarily revert due to out-of-gas, even if you try to forward all available gas to contract `E`.

## 10. What gets returned during an assembly revert?

Using assembly revert lets you return error data more efficiently in terms of gas compared to the Solidity revert.

`revert` in assembly takes two parameters: a memory slot and the size of the data in bytes:

```solidity
revert(startingMemorySlot, totalMemorySize)
```

You control exactly what error data gets returned from an assembly revert. For example, we can choose to revert using the error message returned from a [delegatecall](https://www.rareskills.io/post/delegatecall) by using `returndatasize()` to determine the total memory size of the returned data, like the OpenZeppelin Proxy.sol does:

```solidity!
function _delegate(address implementation) internal {
    assembly {
        calldatacopy(0, 0, calldatasize())

        let result := delegatecall(
            gas(),
            implementation,
            0,
            calldatasize(),
            0,
            0
       )

        returndatacopy(0, 0, returndatasize())
        if iszero(result) {
            revert(0, returndatasize())
        }

        return(0, returndatasize())
    }
}
```

Using low level assembly, let’s simulate Solidity’s revert statements and their return data to get a better understanding of the structure of the returned data in assembly revert.

### Simulating a revert without a reason string

Similar to `revert()` in Solidity, `revert(0,0)` is the equivalent in Inline assembly. It doesn’t return any error data as the starting memory slot is defined to be `0`, and it has a data size of `0`, which indicates that no data should be returned.

```solidity!
contract ContractB {

    function revertWithAssembly() external pure {
        assembly {
            revert(0, 0) // no returned data
        }
    }
}
```

### Simulating a revert with a reason string

Revert with a reason in Solidity — `revert(string)` involves several steps under the hood:

-   ABI encoding of the `Error(string)`
-   allocating memory to store the string metadata like the length and offset
-   and allocating memory for the actual string.

All of these steps can increase gas costs.

To optimize gas cost, you can achieve similar functionality using assembly. This method reduces the steps and opcodes required since we know and control exactly how the data is stored, while still returning the same error data. In the example below, we manually manipulated the memory and directly stored the:

-   The function selector of the `Error(string)` — we can get the selector outside of the contract and just use it. I added the encoding in the example for clarity.
-   The offset
-   The string length
-   The actual string
-   And triggered a revert

```solidity!
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract ContractB {
    function revertwithAssembly() external pure {
        bytes4 selector = bytes4(abi.encodeWithSignature("Error(string)")); //selector with leading zeros

        assembly {
            mstore(0x00, selector) //- Store the function selector for `Error(string)`
            mstore(0x04, 0x20) //- Store the offset to the error message string
            mstore(0x24, 0xc) //- Store the length of the error message string
            mstore(0x44, "Unauthorized") //- Store the actual error message
            revert(0x00, 0x64) //- Trigger the revert revert(StartingMemorySlot, totalMemorySize)
        }
    }
}
```

And when we call the contract from this external contract:

```solidity!
import "hardhat/console.sol";

contract ContractA {
    function call_failure(address contractBAddress)
        external
        returns (bytes memory err)
    {
        (, err) = contractBAddress.call(
            abi.encodeWithSignature("revertwithAssembly()")
        );

        console.logBytes(err);
    }
}
```

The result will be the encoded data in hexadecimal:

![Revert with reason string (revert(reason)) remix output](https://static.wixstatic.com/media/706568_ec719f3e3ee94bbfb600ddafa874b466~mv2.png/v1/fill/w_666,h_366,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_ec719f3e3ee94bbfb600ddafa874b466~mv2.png)

Which is the same as what we got when we used the Solidity `revert(string)`

### Simulating a revert with custom error

We can simulate a custom revert with assembly for the purpose of further saving gas. A custom revert in assembly returns the function selector just like the Solidity custom revert.

However, unlike in Solidity, where the full encoding of the custom error occurs, we can eliminate that step and just store the selector directly and trigger the revert.

```solidity!
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";

contract ContractB {
    function customRevertWithAssembly() external pure {
        bytes32 selector = bytes32(abi.encodeWithSignature("Unauthorized()"));

        assembly {
            mstore(0x00, selector) //- Store the function selector for the custom error

            revert(0x00, 0x04)
        }
    }
}

contract ContractA {
    function call_failure(address contractBAddress)
        external
        returns (bytes memory err)
    {
        (, err) = contractBAddress.call(
            abi.encodeWithSignature("customRevertWithAssembly()")
        );

        console.logBytes(err);
    }
}
```

We should see the selector as the returned value when we run it as shown below:

![revert with custom error remix output](https://static.wixstatic.com/media/706568_2621998f8bdb453caa4be031372fc0dd~mv2.png/v1/fill/w_666,h_587,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_2621998f8bdb453caa4be031372fc0dd~mv2.png)

### Summary of all the ways a Solidity contract can revert

When a transaction reverts via a `require` statement with a reason string or a `revert` containing a string, the return value for the error is the selector of `Error(string)` followed by the ABI encoding of the reason string.

When a transaction reverts due to an `assert` or an illegal operation, the error data is the selector of `Panic(uint256)` followed by the error code ABI encoded as a `uint256`.

The error data is empty when:

-   a transaction reverts with a `require()` or a `revert()` statement with no reason string
-   the called contract uses up the gas
-   the called contract uses assembly and reverts with `revert(0, 0)`

## Part 2: How `try/catch` handle each situation

In the first section of this guide, we’ve seen the different ways different reverts return errors. Now, let's explore how the `try/catch` statement in Solidity handles each of these situations.

The `try/catch` statement provides a structured way to handle exceptions that may occur during external function calls or interactions without reverting and rolling back the entire transaction. However, the state changes in the called contract will still revert if an error occurs.

### The try/catch statement

This is a typical structure of a `try/catch` statement. Note that this is pattern matching all the ways Solidity can revert:

```solidity!
function callContractB() external view {
    try functionFromAnotherContract() {
        //<-- Handle the success case if needed
    } catch Panic(uint256 errorCode) {
        //<-- handle Panic errors
    } catch Error(string memory reason) {
        //<-- handle revert with a reason
    } catch (bytes memory lowLevelData) {
        //<-- handle every other errors apart from Panic and Error with a reason
    }
}
```

The different types of revert we discussed earlier can be caught in different sections of the `try/catch` block based on their returned values.

The `catch Error(string memory reason)` block handles all reverts with a reason string. That means, `revert(string)` and `require(false, “reason”)` errors will be caught here. This is because those errors return the `Error(string)` error when triggered.

The `catch Panic(uint256 errorCode)` will catch all illegal operations, such as dividing by zero at the Solidity level, and `assert` errors as those errors return `Panic(uint256 errorCode)` when triggered.

Finally, any other error that does not return Panic or Error will be caught in the generic `catch block catch (bytes memory lowLevelData)`, including the custom errors and the errors without a message string.

You can also use the `catch{ }` block if you are not interested in the error data. And that will catch any error from the called contract.

Let’s take a look at an example of the `try/catch` syntax. In this simple example, we’ll try to simulate the different types of errors and write a try/catch to handle them based on their error return values.

```solidity!
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract ContractB {

    error CustomError(uint256 balance);

    uint256 public balance = 10;

    function decrementBalance() external {
        require(balance > 0, "Balance is already zero");
        balance -= 1;
    }

    function revertTest() external view {
        if (balance == 9) {
            // revert without a message
            revert();
        }

        if (balance == 8) {
            uint256 a = 1;
            uint256 b = 0;
            // This is an illegal operation and should cause a panic (Panic(uint256)) due to division by zero
            a / b;
        }

        if (balance == 7) {
            // revert with a message
            revert("not allowed");
        }

        if (balance == 6) {
            // revert with a message
            revert CustomError(100);
        }
    }
}
```

Our try/catch block call to handle these errors would look like this:

```solidity!
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

import "hardhat/console.sol";
import {ContractB} from "contracts/revert/contractB.sol";

contract ContractA {

    event Errorhandled(uint256 balance);

    ContractB public contractB;

    constructor(address contractBAddress) {
        contractB = ContractB(contractBAddress);
    }

    function callContractB() external view {

        try contractB.revertTest() {
            // Handle the success case if needed
        } catch Panic(uint256 errorCode) {
            // handle illegal operation and `assert` errors
            console.log("error occurred with this error code: ", errorCode);
        } catch Error(string memory reason) {
            // handle revert with a reason
            console.log("error occured with this reason: ", reason);
        } catch (bytes memory lowLevelData) {
            // revert without a message
            if (lowLevelData.length == 0) {
                console.log("revert without a message occured");
            }

            // Decode the error data to check if it's the custom error
            if (                bytes4(abi.encodeWithSignature("CustomError(uint256)")) ==                bytes4(lowLevelData)
            ) {
                // handle custom error
                console.log("CustomError occured here");
            }
        }
    }
}
```

In each of the catch blocks, we’ve simulated the handling of the different types of error types. For a better understanding, I’ve added comments to explain what is happening in each block.

Notice there is no catch block for custom errors like `catch CustomError {}`. Instead, we handle it in the generic catch-all block with a manual process because there's no official way to decode custom errors yet.

There is an [open issue](https://github.com/ethereum/solidity/issues/11278) about this as well with a lot of suggestions on how you can hack it by adding `if` statements in the final catch block to match the selector. The `try/catch` [release blog](https://soliditylang.org/blog/2020/01/29/solidity-0.6-try-catch/) mentioned that there is a plan to improve the try/catch statement to handle custom errors properly in the future.

![Catch future plans by solidity](https://static.wixstatic.com/media/706568_05bd109324b6499fb792ed026590bbbc~mv2.png/v1/fill/w_666,h_245,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_05bd109324b6499fb792ed026590bbbc~mv2.png)

In our case, we checked if the low-level error data corresponds to a specific custom error signature we are trying to catch.

```solidity!
if (bytes4(abi.encodeWithSignature("CustomError(uint256)")) == bytes4(lowLevelData)
){}
```

## What are the scenarios try/catch will fail to handle your errors?

Since the `try / catch` syntax only catches errors from external contracts:

### 1) Any error that occurs inside a try or catch block (in the calling contract) will not be caught.

For example, none of the reverts in this image will be caught ([run via remix](https://remix.ethereum.org/#activate=solidity,fileManager&gist=a46e9078e8373bec1cbe5a800b9a1a53&call=fileManager//open//a46e9078e8373bec1cbe5a800b9a1a53/uncatchableTryCatch.sol&lang=en&optimize=false&runs=200&evmVersion=null&version=soljson-v0.8.26+commit.8a97fa7a.js)):

![Example of errors that try or catch block will not catch in the calling contract](https://static.wixstatic.com/media/706568_c7af2470171144db81057aed9ede3035~mv2.png/v1/fill/w_666,h_608,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_c7af2470171144db81057aed9ede3035~mv2.png)

### 2) If the contract doesn’t have the right kind of “catch”

For example, If the contract reverts with panic but only has an error catch block and no general catch block, as shown in the screenshot below ([run via Remix](https://remix.ethereum.org/#activate=solidity,fileManager&gist=d0385aba51a85defb784282f7bd2c55f&call=fileManager//open//gist-d0385aba51a85defb784282f7bd2c55f/theRightKindOfCatch.sol)):

![Example of when the contract has the wrong kind of "catch"](https://static.wixstatic.com/media/706568_4dd358bf51f64eb4ab36e3b70be4c716~mv2.png/v1/fill/w_666,h_668,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_4dd358bf51f64eb4ab36e3b70be4c716~mv2.png)

### 3) Revert when an interface expects return data but none is provided

If the interface defining the call to the other contract expects return data and the contract does not return any, or returns it in an unexpected format, the entire transaction will revert and will not be caught, as shown in the example below ([run via Remix](https://remix.ethereum.org/#activate=solidity,fileManager&gist=b14fbe071b4ef64439dc995d72f51032&call=fileManager//open//gist-b14fbe071b4ef64439dc995d72f51032/noReturnValue.sol)):

![Revert when an interface expects return data but none is provided](https://static.wixstatic.com/media/706568_c1b7cc8df2204d5db8a2c92d045c73df~mv2.png/v1/fill/w_666,h_605,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_c1b7cc8df2204d5db8a2c92d045c73df~mv2.png)

## Issues with the `try/catch` in Solidity

The issues with try/catch has been a [topic of discussion](https://forum.soliditylang.org/t/call-for-feedback-the-future-of-try-catch-in-solidity/1497) with a lot of propositions and we’ve seen some of them in the course of this guide.

The issues highlighted in the discussion include:

### Wrong expectations created by the syntax

There is a misconception that the `try/catch` syntax works in Solidity like in other languages, as we’ve already seen, it isn’t the same. For example, you would expect that the following code flow should work.

```solidity!
try <expression> {
   revert();
} catch {
     // also handle the revert in the try block
}
```

But it won’t work as expected. The catch block won’t catch the revert. It will terminate the entire transaction.

### Lack of mechanism to handle reverts in compiler-generated checks

When we make [high-level calls](https://www.rareskills.io/post/low-level-call-solidity) to a function from another contract, the Solidity compiler performs several checks on the called contract, such as:

-   Checking if `extcodesize` of the target contract to check [if](https://www.rareskills.io/post/solidity-code-length) [the target is a contract](https://www.rareskills.io/post/solidity-code-length). It fails if the address is not a contract.
-   Checking `returndatasize` — if the method is expected to return some data, it verifies if `returndatasize` is not empty. If there are returned values, it decodes them and validates they were correctly encoded.
-   It also does encoding and decoding checks. The caller will also try to ABI-decode the data returned, and will revert if the data is malformed or non-existent.

If any of these checks fail, the try/catch syntax will not catch the error.

### Missing features

Aside, `catch Panic(uint256 errorCode)` and `catch Error(string memory reason)` having a feature that allows you to catch custom errors like `catch CustomError()` is a feature that is naturally expected in a try/catch syntax. However, there is no syntax for those error cases, they must be handled manually in the catch block.

## Suggested solutions to the `try/catch` issues in Solidity

From the [proposal discussion](https://forum.soliditylang.org/t/call-for-feedback-the-future-of-try-catch-in-solidity/1497) we mentioned earlier, this is a brief summary of the suggested solutions which are at the time of this writing not implemented:

-   Extending the `try/catch` syntax with additional features that explicitly define the type of error you are handling. For example, the `internal` catch will handle local reverts triggered by extra checks added by the compiler (it still won't catch reverts triggered within the same contract), while the `external` catch will continue to function with the existing catch implementation.  
-   Adding new catch clauses for local reverts like
-   catch NoContract {}
-   catch DecodingFailure {}
-   catch Other {}  
-   `tryCall()` and `match` — this feature is expected to run a pattern match on the external function and allow you to handle then various errors in different arms of the match construct depending on the result and type of error. Here is an example from the [`proposal`](https://forum.soliditylang.org/t/call-for-feedback-the-future-of-try-catch-in-solidity/1497):

```solidity!
import { tryCall } from "std/errors";
error MyError(string);

match tryCall(token.transfer, (exampleAddress, 100)) {
    CallSuccess(transferSuccessful) => {
        ...
    }
    
    CallFailure(MyError(reason)) => {
        ...
    }
    NotAContract => {
        ...
    }
    
    DecodingFailure(errorCode) => {
        ...
    }
}
```

This proposal was posted in February 2023. So, we look forward to the approach that wins and gets implemented in the future.

## Conclusion

When a Solidity contract reverts, it can return an ABI-encoded `Error(string)`, `Panic(uint256)`, a 4-byte custom error, or nothing at all. The try-catch has catches to handle `Error(string)`, `Panic(uint256)`, and a general `catch`. It cannot natively handle custom errors.

Try catch fails if the caller reverts or the callee returns data in a format the caller isn’t expecting (such as trying to parse empty or malformed data).

## Authorship

This article was written by [Eze Sunday](https://www.linkedin.com/in/ezesundayeze/) in collaboration with RareSkills.

*Originally Published July 25, 2024*
