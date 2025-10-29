# Solidity Staticcall EIP 214

Staticcall is like a regular Ethereum call except that it reverts if a state change happens. It cannot be used to transfer Ether. Both the EVM opcode, the Yul assembly function, and the built-in Solidity function named staticcall.

## EIP 214

Staticcall was introduced in [EIP 214](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-214.md) and added to Ethereum in 2017 as part of the [Byzantium hard fork](https://ethereum.org/en/history/#byzantium). For functions where only a return value is desired and state changes are unwanted, it adds a layer of safety. Staticcall will revert if the contract called causes a state change which includes: logging an event, sending Ether, creating a contract, destroying a contract, or setting a storage variable. Reading a storage variable is ok. A call that does not transfer Ether is allowed as long as it doesn't result in a state change.

## View functions

Solidity compiles view functions that call external contracts to use staticcall under the hood.

Here is a minimal example

```solidity
contract ERC20User {
    IERC20 public token;

    // rest of the code

    function myBalance() public view returns (uint256 balance) {
        balance = token.balanceOf(address(this));
    }

    function myBalanceLowLevelEquivalent() public view returns (uint256 balance) {
        (bool ok, bytes memory result) = token.staticcall(abi.encodeWithSignature("balanceOf(address)", address(this)));
        require(ok);

        balance = abi.decode(result, (uint256));

    }
}
```
If `balanceOf` changes the state, the code will revert.

## Meta Arguments

The amount of gas forwarded can be specified in the following manner. The amount of gas forwarded is subject to the 63/64 rule described in [EIP 150](https://www.rareskills.io/post/eip-150-and-the-63-64-rule-for-gas).

```solidity
staticcall{gas: gasAmount}(abiEncodedArguments);
```

## Attack Vectors

Although staticcall is in some sense "safer" than a regular call, that doesn't mean one can neglect security when using it.

### Denial of Service

Staticcall is still subject to denial of service attacks with gas griefing. Consider the situation where the ERC20 token above put an infinite loop inside the balanceOf function. The calling contract will still have gas left over, but only 1/64th of the original amount, per [EIP 150](https://www.rareskills.io/post/eip-150-and-the-63-64-rule-for-gas).

### Read-only reentrancy

Another attack vector for the static call is getting the wrong values back. In the [read-only reentrancy attack](https://www.rareskills.io/post/where-to-find-solidity-reentrancy-attacks), token values are temporarily manipulated with a flashloan, so that any smart contract that is viewing the values (via staticcall) could be hacked with oracle manipulation.

## Staticcall in Yul Assembly

In Yul assembly, the arguments for staticcall are as follows

```solidity
let ok := staticcall(gas, addressToCall, in, insize, out, outsize)
```

The arguments out and outsize are the region in memory where the return value will be copied to. However, both values are typically set to zero and "returndatacopy" and "returndatasize" are used instead to flexibly handle variable return sizes.

These opcodes were introduced in [EIP 211](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-211.md), removing the need to predict the length of the return value from the other smart contract.

See this example:

```solidity
returndatacopy(0, 0, returndatasize())
```
The variables in and insize point to the region in memory where the (usually abi-encoded) data portion of the call to the external contract is.

The variable ok returns true if the staticcall succeeds and false if it reverts. Reverts are not bubbled up.

## Usage with precompiled smart contracts

Staticcall is the appropriate way to interact with [Ethereum precompiled contracts](https://www.rareskills.io/post/solidity-precompiles) (addresses 0x01 to 0x09) as none of them are state changing.

The following example will compute the SHA256 of a uint. Note that this precompile hashes the data directly, and it should not be [ABI-encoded](https://www.rareskills.io/post/abi-encoding).

```solidity
 function getSha256(uint256 x) public view returns (bytes32 hashOut) {
    (bool ok, bytes memory result) = address(0x02).staticcall(abi.encode(x));
    require(ok);

    hashOut = abi.decode(result, (bytes32));
}
```

## Learn more

See our expert [Solidity developer bootcamp](https://www.rareskills.io/solidity-bootcamp) to learn more advanced Ethereum development concepts. We also have [free Solidity course](https://www.rareskills.io/learn-solidity).

*Originally Published Apr 10, 2023*
