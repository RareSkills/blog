# EIP-150 and the 63/64 Rule for Gas

## Introduction

[EIP-150](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md), or Ethereum Improvement Proposal 150, is a protocol upgrade for the Ethereum blockchain. It was proposed on March 18, 2016, and implemented on July 20, 2016, as part of the Ethereum Byzantium hard fork. There were several changes in the protocol, but we will focus on the 63/64 rule it introduced.

Before we delve into the specifics of EIP-150, it is important to understand the concept of gas and contract calls in [Ethereum](https://www.rareskills.io/post/convert-gas-to-usd-ethereum).

### Authorship

This article was co-written by tanim0la ([Twitter](https://twitter.com/tanim0la)) as part of the RareSkills Technical Writing Program.

### Concept of gas in Ethereum

Gas is a unit of measurement for the computational power required to execute a particular operation or contract on the Ethereum Virtual Machine (EVM).

Every operation or contract on the EVM requires a certain amount of gas to be executed, which must be paid for by the user in the form of Ether. Whenever a user initiates a transaction, they are obliged to cover the cumulative costs of all operations performed by their transaction.

For a basic Ethereum transfer, the cost of such operation exactly amounts to 21,000 gas; however, more complex operations may require a significantly greater number of gas units, potentially amounting to millions.

### Contract Calls

In Ethereum, a contract (known as the "caller") has the ability to invoke other contracts (referred to as "callees") through special opcodes such as `CALL`, `STATICCALL` and `DELEGATECALL`. When this occurs, the "callees" receive a specific amount of gas, similar to what they would receive if they were called directly via a transaction.

The quantity of gas allocated is determined in part by the caller, as specified in the opcode parameters. For further information, see the CALL specification provided [here](https://eips.ethereum.org/EIPS/eip-5) as an example. If a callee receives an insufficient amount of gas, its operations are reverted, triggering an "out of gas" exception.

## Specification of EIP-150

Previously, a caller could send all the gas provided to them to the callee. However, this allowed for a potentially endless series of contracts calling other contracts, since the gas cost of each call was low.

In order to prevent the occurrence of a "stack too deep" issue in the Ethereum node's implementation, the maximum depth was limited to 1024, which remains in place today. When this depth is reached, the last call will revert.

As a result, transaction signers had the ability to guarantee that a particular call would revert by having the transaction go through a sequence of calls until it reached a depth of 1023 before calling the desired smart contract. This type of attack is referred to as the "Call Depth Attack".

Here is an example of a contract that would be vulnerable to this attack:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.5;

// DO NOT USE!!!
contract Auction {
    address highestBidder;
    uint256 highestBid;

    function bid() external payable {
        if (msg.value < highestBid) revert();

        if (highestBidder != address(0)) {
            payable(highestBidder).send(highestBid); // refund previous bidder
        }
        highestBidder = msg.sender;
        highestBid = msg.value;
    }
}
```

Prior to the implementation of EIP-150, the contract above could be susceptible to "Call Depth Attack." This is because a malevolent bidder could initiate a recursive call to itself, causing the stack depth to increase to 1023 before calling the bid() function. As a result, the send(highestBid) call would fail silently, meaning that the previous bidder would not receive a refund, while the new bidder would still become the highest bidder.

The solution put forward in [EIP-150](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-150.md) was to reserve a portion of the available gas within the caller contract, after calling the callee's contract (i.e the call cannot consume more than 63/64 of the gas of the parent).

Formula for gas reserved for the parent:

```
Reserved portion of the available gas  = available gas at Stack depth N - ((63/64) * available gas at Stack depth N)
```

Let's test the above formula!

```
Assume; available gas at Stack depth 0 = 1000

Reserved portion of the available gas  = 1000 - ((63/64) * 1000) = 15
```

### The 63/64 Rule: Gas the callee can receive

The formula below calculates the gas available at any stack depth. Also, the gas available is inversely proportional to stack depth (the deeper the stack depth, the lower the gas available).

```
Gas available at Stack depth 0  = Initial gas available * (63/64)^0
Gas available at Stack depth 1  = Initial gas available * (63/64)^1
Gas available at Stack depth 2  = Initial gas available * (63/64)^2
Gas available at Stack depth 3  = Initial gas available * (63/64)^3
.
.
.
Gas available at Stack depth N  = Initial gas available * (63/64)^N
```

**Here are some example values.**

```
Assume; Initial gas available = 3000

Gas available at Stack depth 10  = 3000 * (63/64)^10 = 2562

Gas available at Stack depth 20  = 3000 * (63/64)^20 = 2189
```

![eip 150 ethereum gas at stack call](https://static.wixstatic.com/media/935a00_b4f3701be64e4b54a3e3c585d0e69231~mv2.png/v1/fill/w_666,h_352,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b4f3701be64e4b54a3e3c585d0e69231~mv2.png)

Due to the rapid decrease of gas at each additional depth level, the recursive depth would get limited naturally. Although the limit of stack depth 1024 still exists in the current Ethereum implementation, it is practically unattainable.

Aside from the modification previously mentioned, EIP-150 also introduced a change to the CALL\* opcodes where the gas provided is now a maximum value instead of a strict value. This means that if the available gas in the contract being called (callee) is lower than the specified value given to the opcode, the call will still proceed but with a reduced amount of gas instead of failing, this was not the case in previous versions.

## Conclusions

To summarize, the EIP-150 was introduced to prevent the Call Depth Attack. It does this by enforcing 63/64 rule specification, the implications for this are that even when explicitly forwarding all the [gas left](https://www.rareskills.io/post/solidity-gasleft) in a call, a fraction will still be reserved for the calling contract.

### Learn more

Join our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp) to develop a comprehensive understanding of the EVM and the Ethereum protocol.

*Originally Published March 23, 2023*