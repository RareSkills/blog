# Ethereum smart contract creation code

```evm-bytecode
608060405260405160893803806089833981016040819052601e916025565b600055603d565b600060208284031215603657600080fd5b5051919050565b603f80604a6000396000f3fe6080604052600080fdfea26469706673582212204a131c1478e0e7bb29267fd8f6d38a660b40a25888982bd6618b720d4498b6b464736f6c634300080700330000000000000000000000000000000000000000000000000000000000000001
```

This article explains what happens at the bytecode level when an Ethereum smart contract is constructed and how the constructor arguments are interpreted.

## Table of contents

We discuss the following topics with visual examples:
- Introduction
- Init code
    - Payable constructor contract
    - Non-payable constructor contract
- Runtime code
    - Runtime code breakdown
- Constructor with parameters

## Introduction

At a high level, the wallet deploying the contract sends a transaction to the null address with the transaction data laid out in three parts:

```evm-bytecode
<init code> <runtime code> <constructor parameters>
```

Together they're called the creation code. The EVM starts by executing the init code. If the init code is properly encoded, this execution will store the runtime code on the blockchain.

There is nothing in the EVM specification that says the layout must be init code, runtime code and constructor parameters. It could be init code, constructor parameters, and then runtime code. This is simply the convention that Solidity uses. However, the init code must be the first part for the EVM to know where to begin executing.

**Prerequisites**

This article assumes knowledge of the following topics:
- Solidity (See our [free Solidity tutorial](https://www.rareskills.io/learn-solidity) if you are just starting.).
- Basics of EVM opcodes

Let's dive in!

## Authorship

This article was co-written by Michael Amadi ([LinkedIn](https://www.linkedin.com/in/michael-amadi-2aa2ab23b/), [Twitter](https://twitter.com/@AmadiMichaels)) as part of the RareSkills Technical Writing Program.

## Solidity creationCode

Solidity has a mechanism to get the bytecode that will be deployed during the smart contract creation transaction via the creationCode keyword. This is demonstrated below.

This does not include the constructor arguments, which will be included as part of the bytecode ran during contract deployment. How the init code (creationCode) and arguments are structured is explained in this article.

```solidity
contract ValueStorage {
    uint256 public value;
    constructor(uint256 value_) {
        value = value_;
    }
}

contract GetCreationCode {
    function get() external returns (bytes memory creationCode) {
        creationCode = type(Simple).creationCode;
    }
}
```

## Init code

The init code is the fragment of the creation code responsible for deploying a contract. Let's look at the simplest possible smart contract. We will explain why we added a payable constructor later.

### Payable constructor contract

```solidity
pragma solidity 0.8.17;// optimizer: 200 runscontract Minimal {
    constructor() payable {

    }
}
```

To get the compilation result, we can copy the "input" field from remix after executing the deployment transaction.

![extract contract creation bytecode](https://static.wixstatic.com/media/935a00_25371a89bdbb40228a009c0da2704f5c~mv2.png/v1/fill/w_740,h_399,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_25371a89bdbb40228a009c0da2704f5c~mv2.png)
extract contract creation bytecode

When we copy the highlighted field, we get

```evm-bytecode
0x6080604052603f8060116000396000f3fe6080604052600080fdfea2646970667358221220d03248cf82928931c158551724bebac67e407e6f3f324f930c4cf1c36e16328764736f6c63430008110033
```

This is of course rather hard to read. However, we can break it into two parts.

![Init code and runtime code highlighted](https://static.wixstatic.com/media/935a00_b7ef73f4ed484fea99a315c0efd0f691~mv2.png/v1/fill/w_740,h_240,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b7ef73f4ed484fea99a315c0efd0f691~mv2.png)

It might seem like we split the bytecode in a random place, but it will be explained more clearly later.

If we copy and paste the first part into evm codes, and convert the bytecode to mnemonics, we get the following [output](https://www.evm.codes/playground?fork=merge&unit=Wei&codeType=Bytecode&code=%276080604052603f8060116000396000f3fe%27_). Comments have been added.

```evm-bytecode
// allocate free memory pointer
PUSH1 0x80
PUSH1 0x40
MSTORE

// length of the runtime code
PUSH1 0x3f
DUP1

// where the runtime code begins
PUSH1 0x11
PUSH1 0x00// copy the runtime code from calldata into memory
CODECOPY

// runtime code is deployed at this step
PUSH1 0x00
RETURN
INVALID
```

The section of code highlighted in the visual, referred to as the runtime code, has a size of 63 bytes (0x3f in hexadecimal). It starts at the 17th index (0x11 in hexadecimal) in memory. This explains where the values of 0x3f and 0x11 came from in the mnemonics breakdown above.

On a high level, the following three actions occur within this init code:
- The free memory pointer, which keeps track of the next available memory location for writing, is assigned.
- The runtime code is then copied into this memory location using the "CODECOPY" opcode.
- Finally, the region of memory containing the runtime code is returned to the EVM which stores it as the new contract's runtime bytecode.

### Non-payable constructor contract

```solidity
pragma solidity 0.8.17;// optimizer: 200 runs
contract Minimal {
    constructor() {

    }
}
```

Let's look at the bytecode when the constructor is not payable and see the differences. This is the compiler output.

```evm-bytecode
6080604052348015600f57600080fd5b50603f80601d6000396000f3fe6080604052600080fdfea2646970667358221220a6271a05446e269126897aea62fd14e86be796da8d741df53bdefd75ceb4703564736f6c63430008070033
```

Breaking this down into the init and runtime codes, we have

![Init code and runtime code of a non-payable constructor empty contract](https://static.wixstatic.com/media/935a00_87200a2c332346488c67cc4d575205c7~mv2.png/v1/fill/w_740,h_148,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_87200a2c332346488c67cc4d575205c7~mv2.png)

Let's lay out the payable and non-payable init code side by side.

```evm-bytecode
0x6080604052603f8060116000396000f3fe // payable
0x6080604052348015600f57600080fd5b50603f80601d6000396000f3fe // non-payable
```

We can notice that the payable contract's init code is smaller than that of the non-payable one. We explain why below.

Pasting the longer sequence (non-payable) into evm codes, we get the following [output](https://www.evm.codes/playground?fork=merge&unit=Wei&codeType=Bytecode&code=%27~80~4052348015~0f57z80fd5b50~3f80~1dz39zf3fe%27~60z~00%01z~_), with comments added.

```evm-bytecode
// initialize free memory pointer
PUSH1 0x80
PUSH1 0x40
MSTORE

// check the amount of wei that was sent
CALLVALUE
DUP1
ISZERO

// Jump to 0x0f (contract deployment step)
PUSH1 0x0f
JUMPI

// revert if wei sent is greater than 0
PUSH1 0x00
DUP1
REVERT

// Jump dest (0x0f)
JUMPDEST
POP

// length of the runtime code
PUSH1 0x3f
DUP1

// where the runtime code begins
PUSH1 0x1d
PUSH1 0x00
CODECOPY
PUSH1 0x00
RETURN
INVALID
```

To explain what's happening above we explain and use these concepts:

### Difference between payable and non-payable constructors

**1. The init code reverts if callvalue > 0, otherwise the code will continue its execution.**

The non-payable constructor has an extra byte sequence of 348015600f57 600080fd 5b50 (12 bytes) in between the free memory pointer initialization and returning the runtime code in the non payable contract.

```evm-bytecode
<init bytecode> <extra 12 byte sequence (payable case)> <return runtime bytecode> <runtime bytecode>
```

This additional code checks that during deployment, no value (wei) is sent (sequence 348015600f57) and reverts otherwise (sequence 600080fd). The last two bytes 5b50 are a JUMPDEST and POP opcodes that begin the sequence of deployment described earlier if no wei was sent.

(The reason there is a pop is because the callvalue is still on the stack and we no longer need it. A JUMPDEST is simply a target for JUMPs and JUMPIs. Without it at a specified jump location the JUMPs can't land and will revert.)

**2. The memory offsets for the runtime code are shifted**

Also notice that the length of the runtime code doesn't change but the offset to copy the runtime code does change because the init code is longer which shifts the runtime code's offset further down.

The non-payable offset for the init bytecode is 0x1d, and the offset for the payable case is smaller at 0x11. If we subtract them (0x1d - 0x11 = 0x0c, 12 in decimal) we get the size of extra byte sequence checking for non-zero value in between the free memory pointer initialization chunk and the sequence where the runtime bytecode is returned.

## Runtime code for an empty contract

### The runtime code is non-empty in an empty contract because of the metadata the compiler adds

The runtime code is the fragment of the creation code that is returned by the init code and is set to be the bytecode of the contract that is callable by users after deployment. It becomes what we know of as the "smart contract".

A question arises, "if the contract is empty (has no functions), why is the runtime code non-empty?"

The [Solidity](https://www.rareskills.io/solidity-bootcamp) compiler appends some metadata about your contract to the runtime code. More info about contract metadata [here](https://playground.sourcify.dev/). The opcode fe INVALID, is prepended to the metadata to prevent it from being executed.

(The new Solidity version [0.8.18](https://github.com/ethereum/solidity/releases/tag/v0.8.18) adds a compiler setting --no-cbor-metadata where you can tell the compiler not to append this metadata to your contract's bytecode)

### In a pure Yul contract the compiler does not add metadata by default

If the contract was written in pure Yul, there would be no metadata. However, the metadata section can be added, by including .metadata in the global object.

```yul
// the output of the compilation of this contract// will have no metadata by default
object "Simple" {
    code {
        datacopy(0, dataoffset("runtime"), datasize("runtime"))
        return(0, datasize("runtime"))
    }

    object "runtime" {

        code {
            mstore(0x00, 2)
            return(0x00, 0x20)
        }
    }
}
```

The compiler [output](https://www.evm.codes/playground?fork=merge&unit=Wei&codeType=Bytecode&code=%27zd~39~~f3fe%27z0z~600%01z~_) is as follows 6000600d60003960006000f3fe and when converted to mnemonics, we get

```evm-bytecode
// copy runtime code to memory
PUSH1	00
PUSH1	0d
PUSH1	00
CODECOPY

// Returning a zero sized region because there is no runtime code
PUSH1	00
PUSH1	00
RETURN
INVALID
```

In this case, the area in memory returned is zero, because there is no runtime code or metadata.

(The compiler begins at 0x0d and copies 0x00 bytes of runtime code into memory starting at offset 0x00. Then returns 0x00 bytes.)

## Runtime code for a non-empty contract

Now let's add the simplest possible logic to the contract.

``` solidity
pragma solidity 0.8.7;contract Runtime {
    address lastSender;
    constructor () payable {}

    receive() external payable {
        lastSender = msg.sender;
    }
}
```

The output creation code is

```evm-bytecode
608060405260578060116000396000f3fe608060405236601c57600080546001600160a01b03191633179055005b600080fdfea2646970667358221220e9b731ab28726d97cbf5219f1e5eaec508f23254c60b15ed1d3456572547c5bf64736f6c63430008070033.
```

This can be separated as

![init code and runtime code of a non-empty smart contract](https://static.wixstatic.com/media/935a00_68b8fbe1bbfa4712bdfba5ca1ab2759a~mv2.png/v1/fill/w_740,h_212,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_68b8fbe1bbfa4712bdfba5ca1ab2759a~mv2.png)

Let's look at the runtime code in detail

Since this is a Solidity contract we can divide this into the executable bytecode and contract metadata as explained earlier

```evm-bytecode
Runtime code := 0x608060405236601c57600080546001600160a01b03191633179055005b600080fdfe

Metadata := 0xa2646970667358221220e9b731ab28726d97cbf5219f1e5eaec508f23254c60b15ed1d3456572547c5bf64736f6c63430008070033a2646970667358221220e9b731ab28726d97cbf5219f1e5eaec508f23254c60b15ed1d3456572547c5bf64736f6c63430008070033.
```

Let's dive into what the runtime code does using an evm codes [output](https://www.evm.codes/playground?fork=merge&unit=Wei&codeType=Bytecode&code=%27~80~405236~1c57z08054z1z1~a01b03191633179055005bz080fdfe%27~60z~0%01z~_). It's been divided to simplify it.

First we initialize the free memory pointer.

```evm-bytecode
[00] PUSH1      80
[02] PUSH1      40
[04] MSTORE
```

Here we check if data was sent with the transaction, if so we JUMP to program counter (PC) 0x1c where we revert. The only two valid ways for a contract to receive data is regular functions and the fallback. We only have a receive function, so there is no valid way for the contract to receive calldata.

```evm-bytecode
[05] CALLDATASIZE
[06] PUSH1      1c
[08] JUMPI
```

And then we have the code that stores msg.sender.

```evm-bytecode
[09] PUSH1      00
[0b] DUP1
[0c] SLOAD
[0d] PUSH1      01
[0f] PUSH1      01
[11] PUSH1      a0
[13] SHL
[14] SUB
[15] NOT
[16] AND
[17] CALLER
[18] OR
[19] SWAP1
[1a] SSTORE
[1b] STOP
```

This is JUMPDEST 0x1c for the case where calldata was sent. The transaction reverts.

```evm-bytecode
[1c] JUMPDEST
[1d] PUSH1      00
[1f] DUP1
[20] REVERT
[21] INVALID
```

## Constructor with parameters

Contracts with constructor arguments are encoded a little differently. The constructor parameters are expected to be appended at the end of the creation code (after the runtime code) and [ABI encoded](https://www.rareskills.io/post/abi-encoding).

Solidity in particular adds an extra check to ensure the constructor parameter length is at least the length of the expected constructor arguments, else it reverts.

Let's see a simple example. We don't include any runtime code for simplicity. The only code we include is in the constructor, which is not part of the runtime code.

```solidity
// optimizer: 200contract MinimalLogic {
    uint256 private x;
    constructor (uint256 _x) payable {
        x =_x;
    }
}
```

The creation code is

```evm-bytecode
608060405260405160893803806089833981016040819052601e916025565b600055603d565b600060208284031215603657600080fd5b5051919050565b603f80604a6000396000f3fe6080604052600080fdfea26469706673582212204a131c1478e0e7bb29267fd8f6d38a660b40a25888982bd6618b720d4498b6b464736f6c63430008070033
```

Breaking this down we get

```evm-bytecode
"Init code": 0x608060405260405160893803806089833981016040819052601e916025565b600055603d565b600060208284031215603657600080fd5b5051919050565b603f80604a6000396000f3fe"Runtime code (metadata only)": 0x6080604052600080fdfea26469706673582212204a131c1478e0e7bb29267fd8f6d38a660b40a25888982bd6618b720d4498b6b464736f6c63430008070033"Constructor arguments are missing!"
```

Executing the creation code this way would revert in the init code because it expects at least 32 bytes after the runtime code to use as `uint256 _x`. We will see this in more detail when breaking down each opcode. For now we can append the creation code as ABI encoded `uint256(1)` to use as `_x`.

Now, the corrected bytecode

```evm-bytecode
608060405260405160893803806089833981016040819052601e916025565b600055603d565b600060208284031215603657600080fd5b5051919050565b603f80604a6000396000f3fe6080604052600080fdfea26469706673582212204a131c1478e0e7bb29267fd8f6d38a660b40a25888982bd6618b720d4498b6b464736f6c634300080700330000000000000000000000000000000000000000000000000000000000000001
```

Let's analyse this using the evm codes [output](https://www.evm.codes/playground?fork=merge&unit=Wei&codeType=Bytecode&code=%27tw51z893x3xz8983398101w819052z1e91z25vu55z3dvuz208284031215z3657s5b5051919050vz3fxz4au39uf3fetsfea26469706673582212204a131c1478e0e7bb29267fd8f6d38a6zb40a25888982bd6618b720d4498b6b464736f6c6343~x70033yyyyyyy1%27~000z60y~~~x80wz40v565bu6~tzxw52suxfd%01stuvwxyz~_)

**Step 1: Initialize free memory pointer**

As usual with Solidity contracts, we initialize the free memory pointer with `6080604052`.

**Step 2: Get the constructor parameter's length**

```evm-bytecode
// 6040 51 6089 38 03
PC   OPCODE

[05] PUSH1 40
[07] MLOAD
[08] PUSH1 89
[0a] CODESIZE
[0b] SUB
```

Here PUSH1 40 MLOAD MLOADs the free memory pointer to use later. We push the creation code length (without the constructor params) with PUSH1 89 then call CODESIZE (this includes the constructor parameter). We subtract the two to get the length of the constructor parameters.

**Step 3: Copy the constructor paramter to memory**

```evm-bytecode
// 80 6089 83 39
PC   OPCODE

[0c] DUP1
[0d] PUSH1 89
[0f] DUP4
[10] CODECOPY
```

Here we prepare the stack for CODECOPY. We duplicate the subtraction result from above with opcode DUP1 and push 0x89 (length of the creation code without constructor args) to the stack with PUSH1 89. Finally we use DUP4, to bring the memory offset to the top of the stack. Now we call CODECOPY to copy the constructor parameter to memory at the free memory pointer.

**Step 4: Update free memory pointer**

After writing to the code to memory, Solidity updates the free memory pointer as follows.

```evm-bytecode
// 81 01 6040 81 90 52
PC   OPCODE

[11] DUP2
[12] ADD
[13] PUSH1 40
[15] DUP2
[16] SWAP1
[17] MSTORE
```

We do that here by adding the constructor parameter length (0x20) we duplicated earlier to the free memory pointer (0x80) then arranging it with Dup1 and Swap1 operations before calling MSTORE 40 which stores the new value (0xa0) as the free memory pointer.

Next we have a series of dynamic operations and JUMPs that don't execute sequencially but rather based on some conditions. Let's dig deeper.

The steps are numbered so you can follow them sequencially without looking for the required JUMPDEST.

You can use the [playground link](https://www.evm.codes/playground?fork=merge&unit=Wei&codeType=Bytecode&code=%27tw51z893x3xz8983398101w819052z1e91z25vu55z3dvuz208284031215z3657s5b5051919050vz3fxz4au39uf3fetsfea26469706673582212204a131c1478e0e7bb29267fd8f6d38a6zb40a25888982bd6618b720d4498b6b464736f6c6343~x70033yyyyyyy1%27~000z60y~~~x80wz40v565bu6~tzxw52suxfd%01stuvwxyz~_) for this bytecode to try it out yourself too.

**Step 5: Jump to SSTORE's JUMPDEST**

```evm-bytecode
// 601e 91 6025 56
PC   OPCODE

[18] PUSH1 1e
[1a] SWAP2
[1b] PUSH1 25
[1d] JUMP // jump to JUMPDEST 0x25
```

We want to jump to the PC that performs the operations to store the constructor parameter in storage.

We push 1e to the stack. 1e is the position in the program counter where the SSTORE is actually executed, but first we have to check that the constructor parameter copied is at least 32 bytes. This operation begins at program counter 0x25, which is the JUMPDEST above.

**Step 8: Stores the constructor arg in storage slot 0**

This is JUMPDEST 0x1e, JUMPDEST 0x25 executes first and is below. Note that this is step 8, and the previous section was step 5. It is only executed if the conditions in 6 and 7 are completed successful. We introduce it here out of order to maintain the same sequence of the compiled bytecode.

```evm-bytecode
// 5b 6000 55 603d 56
PC   OPCODE

[1e] JUMPDEST
[1f] PUSH1 00
[21] SSTORE
[22] PUSH1 3d
[24] JUMP
```

Here we push 0x00 which is the storage slot we are going to store `_x` at and call SSTORE. Then push the Jump destination for the final codecopy and return.

**Step 6: Checks if the constructor param size is at least 32 bytes**

This is JUMPDEST 0x25

```evm-bytecode
// 5b 6000 6020 82 84
PC   OPCODE

[25] JUMPDEST
[26] PUSH1 00
[28] PUSH1 20
[2a] DUP3
[2b] DUP5

// continue// 03 12 15 6036 57
[2c] SUB
[2d] SLT
[2e] ISZERO
[2f] PUSH1 36
[31] JUMPI // Jump to 0x36 if ISZERO returns 1// else continue and revert// 6000 80 fd

[32] PUSH1 00
[34] DUP1
[35] REVERT
```

Here we check that the constructor param is at least 32 bytes.

First, we push 0x00 to the stack (for later), push the minimum acceptable length 0x20(32 bytes) to the stack. Next we can get the length of the constructor parameter to be compared by checking it's offset and the current free memory pointer which we pushed to the stack earlier. So we use DUP3 to get the offset to the stack and then DUP5 to get the current free memory pointer to the top of the stack.

Calling SUB subtracts and pushes the length to the stack. Now we can directly call SLT (signed less than) to check if it's up to 32 bytes and push 0 if false and 1 if true, the ISZERO opcode checks if the top of the stack (SLT result) is 0, pops it off and pushes the boolean result to the stack, we push the next JUMP location to the stack and jump to it if ISZERO returned 1., else we revert to avoid executing with invalid calldata.

**Step 7: Loads param to stack and arrange the stack for storing the constructor parameter to storage**

This is JUMPDEST 0x36

```evm-bytecode
// 5b 50 51 91 90 50 56
PC   OPCODE

[36] JUMPDEST
[37] POP
[38] MLOAD
[39] SWAP2
[3a] SWAP1
[3b] POP
[3c] JUMP // jump to 0x1e
```

Here we pop off 0 (program counter 26 from step 6) as we don't need it anymore. we MLOAD the constructor param to the stack and clear out the constructor params memory offset as it's also not needed anymore.

**Step 9: Copies the runtime code to memory and returns it**

This is JUMPDEST 0x3d, JUMPDEST 0x1e executes first above

```evm-bytecode
// 5b 603f 80 604a 6000 39 6000 f3 fe
PC   OPCODE

[3d] JUMPDEST
[3e] PUSH1 3f
[40] DUP1
[41] PUSH1 4a
[43] PUSH1 00
[45] CODECOPY
[46] PUSH1 00
[48] RETURN
[49] INVALID

// Unexecutable code (contract metadata)0x6080604052600080fdfea26469706673582212204a131c1478e0e7bb29267fd8f6d38a660b40a25888982bd6618b720d4498b6b464736f6c63430008070033
```

Here we return the contract runtime code as usual from memory.

**Memory just before RETURN executes**

0x00 to 0x40 is the (empty) runtime code and metadata bytecode. 0x40 contains the free memory pointer. 0x80 contains the constructor argument, uint256(1).

```evm-bytecode
0x00<->0x20 = 0x6080604052600080fdfea26469706673582212208f9ffa7a3ab43f0ff61d30330x20<->0x40 = 0x624bf0e9d398f9a91213656b13d9ffc8fd90fdbc64736f6c63430008070033000x40<->0x60 = 0x00000000000000000000000000000000000000000000000000000000000000a00x60<->0x80 = 0x00000000000000000000000000000000000000000000000000000000000000000x80<->0xa0 = 0x0000000000000000000000000000000000000000000000000000000000000001
```

## Conclusion

Smart contract deployment includes a couple of low level operations abstracted away by most languages. We learned how smart contracts are executed using a creation code sent to the null address, the different parts of this creation code, their roles when deploying a contract and how they work together. We saw how constructor arguments are stored, validated, and used to set up the contract.

## RareSkills Blockchain Bootcamp

Please see our advanced [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) offerings to learn more about the expert-level developer training we offer.

*Originally Published Feb 6, 2023*
