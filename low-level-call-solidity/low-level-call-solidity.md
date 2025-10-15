# Low Level Call vs High Level Call in Solidity

A contract in Solidity can call other contracts via two methods: through the contract interface, which is considered a high-level call, or by using the `call` method, which is a low-level approach.

Despite both methods using the **`CALL`** opcode, Solidity handles them differently.

In this article, we'll compare the two: why a low-level call never reverts, whereas a high-level call can revert, and why a low-level call to an empty address is considered successful, while a high-level call reverts when calling a non-existent contract.

## Why does a low-level call (or a delegatecall) never reverts but a call via the contract interface can revert

Before explaining why, let me quote the Solidity documentation that addresses this issue.

> When exceptions happen in a sub-call, they “bubble up” (i.e., exceptions are rethrown) automatically unless they are caught in a [try/catch statement](https://www.rareskills.io/post/try-catch-solidity). Exceptions to this rule are send and the low-level functions call, [delegatecalll](https://www.rareskills.io/post/delegatecall) and [staticcall](https://www.rareskills.io/post/solidity-staticcall): they return false as their first return value in case of an exception instead of “bubbling up”.

Below we show both a high-level call and a low-level **`call`** to compare the behavior. I will employ the **`call`** method in the example below, but the same principles can be extended to **`delegatecall`**.

`Caller` can call `ops()` in `Called` in two ways. Note that `ops()` always reverts:

```solidity!
pragma solidity ^0.8.0;

contract Caller {

    // first call to ops()
    function callByCall(address _address) public returns (bool success) {
        (success, ) = _address.call(abi.encodeWithSignature("ops()"));
    }

    // second call to ops()
    function callByInterface(address _address) public {
        Called called = Called(_address);
        called.ops();
    }
}
contract Called {

    // ops() always reverts
    function ops() public {
        revert();
    }
}
```

**Despite both methods being used to call the same function, and both methods using the opcode** **`CALL`**, **the solidity compiler generates bytecode** to handle the failure cases in different ways. Executing both functions within the `Caller` contract will reveal that `Caller.callByInterface` will revert, while `Caller.callByCall` will not.

At the EVM level, the **`CALL`** opcode returns a boolean indicating whether the call was successful or not, and places this return on the stack. The opcode itself doesn't trigger the revert.

When the call is made via the contract’s interface, Solidity handles this return value for us. It explicitly checks whether the return value is false and initiates a revert unless the call was made within a `try/catch` block.

However, when using low-level calls, we need to manually handle this return boolean and explicitly trigger a revert if desired.

```solidity!
contract Caller {

      //...
      function callByCall(address address) public returns (bool success) {
        (success, ) = address.call(abi.encodeWithSignature("ops()"));
        if (!success) {
            revert("Something went wront");
        }
    }
    //...
}
```

This difference between a high-level call and a low-level call is illustrated in the figure below.

![Low level call handling a revert vs a high-level call handling a revert](https://static.wixstatic.com/media/935a00_e48b1ce51e9f40a3a1641c1e7d0009d9~mv2.png/v1/fill/w_455,h_514,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_e48b1ce51e9f40a3a1641c1e7d0009d9~mv2.png)

## The difference between call and call by interface when calling an empty address

Solidity's low-level **`call`** method doesn't perform a prior check to verify whether the called address corresponds to a contract. The contract can [check if the address is a smart contract](https://www.rareskills.io/post/solidity-code-length) using **`EXTCODESIZE`**, which is the opcode behind the scenes for **`address.code.length`**. If the size is zero, it indicates that there's no contract deployed at that address. However, the `call` method doesn't incorporate this check; it directly executes the **`CALL`** opcode regardless.

When using the interface, checks the target’s code size. In other words, in the bytecode generated for the `callByInterface` function, the **`EXTCODESIZE`** opcode is executed at the specified address before executing the **`CALL`** opcode. If the size returned by **`EXTCODESIZE`** is zero, indicating that there's no contract at that address, the function reverts before executing the **`CALL`** opcode. This explains why the `callByInterface` function reverts if executed with a non-existent contract address, while `callByCall` does not.

This difference between how a low-level call and a high-level call interacts with an empty contract is illustrated below.

![low-level call calling an empty contract vs a high-level call calling an empty contract](https://static.wixstatic.com/media/935a00_40eac90b8e0f4d72be51f31720c970c3~mv2.png/v1/fill/w_494,h_630,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_40eac90b8e0f4d72be51f31720c970c3~mv2.png)


Fundamentally, an execution can revert if it encounters a **`REVERT`** opcode, runs out of gas, or attempts something prohibited, such as dividing by zero. When a call is made to an empty address, none of the above conditions can occur.

## Learn More with RareSkills

See our free [Solidity course](https://www.rareskills.io/learn-solidity) if you are new to Solidity. If you are a more experienced Solidity programmer, please see our advanced [Solidity bootcamp](https://www.rareskills.io/solidity-bootcamp).

## Authorship

This article was written by [João Paulo Morais](https://www.linkedin.com/in/jpmorais/) in collaboration with RareSkills.


 *Originally Published May 1, 2024*
