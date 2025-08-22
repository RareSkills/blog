# How Ethereum address are derived (EOAs, CREATE, and CREATE2)

On Ethereum, smart contracts can be deployed in one of three ways:

1. An Externally Owned Account (EOA) initiates the transaction where the `to` field is set to `null`, and the `data` field contains the contract’s initialization code.
2. A smart contract calls the `CREATE` opcode.
3. A smart contract calls the `CREATE2` opcode.

In this article, we will explore how to predict the address of the contract that will be created in each of these situations.

## Predicting smart contract addresses deployed by an EOA or CREATE

For contracts deployed by an EOA or CREATE opcode (not CREATE2), the address is computed from the Keccak-256 hash of the [RLP-encoded](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/) `sender` address and `nonce`. The resulting contract address is the last 20 bytes (160 bits) of that hash.

$\texttt{address} = \texttt{keccak256}\left( \texttt{RLP}([\texttt{deployer}, \texttt{nonce}]) \right)\texttt{[:20]}$

As shown in the equation above, this method of address calculation **only** depends on the deployer’s address and their nonce. It does **not** consider the contract’s bytecode, constructor arguments, or anything else.

### Recursive Length Prefix (RLP)

At a high level, [RLP](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/) concatenates the data items that are being sent. Each item, except single bytes in the range [0x00, 0x7f], is prefixed with one or more bytes that indicate whether the item is a string or a list, and the length of its payload. Interested readers can consult the documentation linked above.

To see how RLP encoding is used in contract address prediction, let us walk through a practical example.

### RLPDemo Example

In `RLPDemo` contract below, the `predictContractAddress` function implements the same logic as the CREATE opcode's address derivation. It computes the expected deployment address by applying RLP encoding to the sender’s address and nonce.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract RLPDemo {
    // Function to predict the address of a contract that would be deployed by a given address
    function predictContractAddress(
        address deployer,
        uint nonce
    ) public pure returns (address) {
        // This implements the same logic as the CREATE opcode's address derivation

        // For the CREATE opcode, the address is derived as:
        // keccak256(rlp([sender_address, sender_nonce]))
        bytes memory rlpEncoded;

        // RLP encoding rules:
        // - For nonce = 0, the RLP encoding is [0x80] (empty byte string)
        // - For nonce = 1 to 127, the RLP encoding is the single byte itself (0x01 to 0x7f)
        // - For nonce = 128 to 255, the RLP encoding is [0x81, nonce]
        //   where 0x81 indicates a single-byte length prefix for the following byte

        // Note: Full RLP spec supports encoding arbitrary-length integers using a dynamic length prefix,
        // but this function only supports nonces up to 255.

        if (nonce == 0) {
            // For nonce = 0
            rlpEncoded = abi.encodePacked(
                bytes1(0xd6), // RLP prefix for a list
                bytes1(0x94), // RLP prefix for a 20-byte address
                deployer, // 20 bytes of the deployer's address
                bytes1(0x80) // RLP encoding for the nonce 0 is 0x80
            );
        } else if (nonce < 128) {
            // For nonce = 1-127
            rlpEncoded = abi.encodePacked(
                bytes1(0xd6), // RLP prefix for a list
                bytes1(0x94), // RLP prefix for a 20-byte address
                deployer, // 20 bytes of the deployer's address
                uint8(nonce) // Single byte for nonce
            );
        } else if (nonce < 256) {
            // For nonce = 128-255
            rlpEncoded = abi.encodePacked(
                bytes1(0xd7), // RLP prefix for a list (one byte longer)
                bytes1(0x94), // RLP prefix for a 20-byte address
                deployer, // 20 bytes of the deployer's address
                bytes1(0x81), // RLP prefix for a single byte
                uint8(nonce) // The nonce as a single byte
            );
        } else {
            revert("Nonce too large for this demo");
        }

        bytes32 hash = keccak256(rlpEncoded);
        return address(uint160(uint256(hash)));
    }
}
```

To verify that `predictContractAddress` works as intended, we used the EOA `0x17F6AD8Ef982297579C203069C1DbfFE4348c372` to deploy `RLPDemo`  (the same contract above), which resulted in the contract address `0xE2DFC07f329041a05f5257f27CE01e4329FC64Ef`. 

The deployment result described above is shown on the right side of the image below:

![predicted smart contract address compared with actual](https://r2media.rareskills.io/EthereumAddressPrediction/image01.png)

As shown on the left side of the image above, we called `predictContractAddress` with the deployer's address `0x17F6AD8Ef982297579C203069C1DbfFE4348c372` and nonce `0`, and correctly predicted the contract address that was deployed earlier: `0xE2DFC07f329041a05f5257f27CE01e4329FC64Ef`.

Next, let’s examine how the `nonce` is interpreted for both externally owned accounts (EOAs) and contract accounts. 

### Nonce sequence during account deployment

Let’s start by understanding how nonce is defined in Ethereum. According to the [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf), the nonce ****of an account is defined as:

> nonce: A scalar value equal to the number of transactions sent from this address or, in the case
of accounts with associated code, the number of contract-creations made by this account. For account of address a in state $σ$, this would be formally denoted $σ[a]_n$.
> 

From this definition, the nonce is a value attributed to an address that initiates a transaction or deploys a contract. Since EOA can initiate and sign transactions directly, the nonce count can reflect transactions like ETH/token transfers, contract calls, and contract deployments. Importantly, the nonce increments even if the transaction reverts. A reverted transaction is still included in the block, and this counts towards incrementing the nonce.

In contrast, smart contracts cannot initiate transactions on their own; they only execute when invoked by an EOA or another contract. Hence, the nonce for contract account reflects only contract-creation initiated by the contract.

**Note:** *Internal calls, message calls, events, and other operations that happen within transactions are never used to increment the nonce count of an account.*

**Now, let’s look at how nonces are initialized and used to predict EOA and contract accounts.**

New EOA’s nonce value start at `0`, and the value increments by `1` with each transaction. If the new EOA deploys a contract, `0` will be used as the nonce to predict the address. However, if the account has already sent transactions, such as Ether transfers or previous deployments, the nonce will be greater than `0`.

**For a contract account**, the nonce is initialized with **`1`** upon creation, as specified in [EIP-161](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-161.md). A contract’s nonce is incremented by `1` when it creates other contracts using `CREATE`or `CREATE2`.

**For example, suppose Contract A is just deployed.** 

- At that moment, Contract A’s nonce is set to **`1`**. If **Contract A** proceeds to create another contract, say, **Contract B,** this creation will use **nonce = 1** to compute Contract B’s address.
- Once the creation of Contract B is complete, Contract A’s nonce is incremented to **`2`.**
- Suppose **Contract A** wants to create another contract—say, **Contract C**. It will use **`nonce = 2`** for this deployment. After the creation of Contract C, Contract A’s nonce becomes **`3`**, and so on.
- Contract B and contract C, like any new contract, also starts with `nonce = 1`.

### How to get the nonce of an account

There is no EVM opcode to get the nonce of an account. However, the `eth_getTransactionCount` RPC method returns the account nonce for the given account, as described above.

This method returns the [number of transactions](https://viem.sh/docs/actions/public/getTransactionCount#gettransactioncount) sent from the specified address, which corresponds to the account's nonce. For EOAs, this includes ETH/token transfers, contract calls, and contract deployments. For smart contracts, `eth_getTransactionCount` reflects the number of contract creations by a contract address.

The image below shows how only contract deployment increases the `eth_getTransactionCount` nonce for a contract address.

![how deployment transaction increments the nonce for the smart contract](https://r2media.rareskills.io/EthereumAddressPrediction/image02.png)

Here is an example of how to get nonce using the `eth_getTransactionCount` method in JavaScript.

```tsx
// NECESSARY IMPORTS
import { createPublicClient, http } from 'viem';
import { mainnet } from 'viem/chains';

// CREATE A PUBLIC CLIENT
const publicClient = createPublicClient({
  chain: mainnet,
  transport: http()
});

// GET TRANSACTION COUNT (NONCE)
const transactionCount = await publicClient.getTransactionCount({
  address: '0xYourContractAddress'
});
console.log(transactionCount);
```

For testing, we can use the `vm.getNonce` cheatcode in Foundry.

### The Foundry  `getNonce` method

In Foundry, the `vm.getNonce` [cheatcode](https://book.getfoundry.sh/cheatcodes/get-nonce) allows us to retrieve the current nonce of a given account or wallet on the EVM. 

Here are the `getNonce` methods available in the Foundry environment:

```solidity
// Returns the nonce of a given account.
function getNonce(address account) external returns (uint64);
```

In the `test_eoaAndContractNonces()` shown below, we assert that the nonce of an EOA(`userEOA`) starts at `0`,  and that the nonce of a newly deployed contract, `SomeContract` starts at `1`, as expected.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";

contract SomeContract {
    // Could have logic here if needed
}

contract CreateAddrTest is Test {
    address userEOA = address(0xA11CEB0B);
    SomeContract public newContract;

    function setUp() public {
        // Fund the EOA with 10 ether
        vm.deal(userEOA, 10 ether);

        // Deploy SomeContract which will deploy Dummy in its constructor
        newContract = new SomeContract();
    }

    function test_eoaAndContractNonces() public view{
        // 1. EOA nonce should be 0 initially
        uint256 eoaNonce = vm.getNonce(userEOA);
        console.log("EOA nonce:", eoaNonce);
        assertEq(eoaNonce, 0);

        // 2. Contract nonce should be 1
        uint256 contractNonce = vm.getNonce(address(newContract));
        console.log("SomeContract contract nonce:", contractNonce);
        assertEq(contractNonce, 1);
    }
}
```

Terminal result:

![screenshot of terminal after running the test showing the nonces](https://r2media.rareskills.io/EthereumAddressPrediction/image03.png)

### Predicting contract address deployed by an EOA (using LibRLP)

[Solady](https://github.com/Vectorized/solady) provides a utility called [LibRLP](https://github.com/Vectorized/solady/blob/main/src/utils/LibRLP.sol), which includes a `computeAddress` [function](https://github.com/Vectorized/solady/blob/dcdfab80f4e6cb9ac35c91610b2a2ec42689ec79/src/utils/LibRLP.sol#L46-L87) that uses its internal RLP encoding implementation to compute the address. This helper abstracts away the encoding details and directly returns the contract address that would be generated by an EOA or `CREATE` deployment.

```solidity
 function computeAddress(address deployer, uint256 nonce)
        internal
        pure
        returns (address deployed)
    {
        /// @solidity memory-safe-assembly
        assembly {
            for {} 1 {} {
                // The integer zero is treated as an empty byte string,
                // and as a result it only has a length prefix, 0x80,
                // computed via `0x80 + 0`.

                // A one-byte integer in the [0x00, 0x7f] range uses its
                // own value as a length prefix,
                // there is no additional `0x80 + length` prefix that precedes it.
                if iszero(gt(nonce, 0x7f)) {
                    mstore(0x00, deployer)
                    // Using `mstore8` instead of `or` naturally cleans
                    // any dirty upper bits of `deployer`.
                    mstore8(0x0b, 0x94)
                    mstore8(0x0a, 0xd6)
                    // `shl` 7 is equivalent to multiplying by 0x80.
                    mstore8(0x20, or(shl(7, iszero(nonce)), nonce))
                    deployed := keccak256(0x0a, 0x17)
                    break
                }
                let i := 8
                // Just use a loop to generalize all the way with minimal bytecode size.
                for {} shr(i, nonce) { i := add(i, 8) } {}
                // `shr` 3 is equivalent to dividing by 8.
                i := shr(3, i)
                // Store in descending slot sequence to overlap the values correctly.
                mstore(i, nonce)
                mstore(0x00, shl(8, deployer))
                mstore8(0x1f, add(0x80, i))
                mstore8(0x0a, 0x94)
                mstore8(0x09, add(0xd6, i))
                deployed := keccak256(0x09, add(0x17, i))
                break
            }
        }
    }
```

To understand how this works in practice, we will deploy the `CreateAddressPredictor` contract shown below. We will then call `addrWithLibRLP` to test if the computed result is the same as the deployed `CreateAddressPredictor` address. 

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

// Importing LibRLP, which contains the computeAddress function shown above.
import {LibRLP} from "contracts/LibRLP.sol";

contract CreateAddressPredictor {
    // contract embeds Solady’s address computation logic and exposes it through addrWithLibRLP.
    function addrWithLibRLP(
        address _deployer,
        uint256 _nonce
    ) public pure returns (address deployed) {
        return LibRLP.computeAddress(_deployer, _nonce);
    }
}
```

Using the EOA `0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2`, we deployed `CreateAddressPredictor` on Remix at address `0xa131AD247055FD2e2aA8b156A11bdEc81b9eAD95`.

Here is the terminal result.

![remix result showing predicted vs actual address](https://r2media.rareskills.io/EthereumAddressPrediction/image04.png)

When we call `addrWithLibRLP`, passing in the same EOA used to deploy `CreateAddressPredictor` and a nonce of `0`, the returned address matches the actual deployed address, as expected.

As seen in the image below, the actual deployed contract address matches this predicted address.

![remix screenshot showing derived address vs actual address](https://r2media.rareskills.io/EthereumAddressPrediction/image05.png)

Note: If the nonce is set to any non-zero value in this example, the decoded output will return an incorrect address, because we are testing this with a fresh EOA account.

### Predicting a contract address deployed by a contract

As noted earlier, the address derivation for a deployed contract is the same whether the deployer is an EOA or a contract. We only need to set the deployer address and nonce correctly. 

In the test below, the `Deployer` contract demonstrates how the address returned from the `computeAddress` method correspond to the contract deployed by another contract.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "contracts/LibRLP.sol";

contract C {}

contract Deployer {
    // Note: nonce is not stored on-chain — this is just for tracking purposes
    uint256 public contractNonce = 1;

    function deploy() public returns (address c) {
        address predicted = predictAddress(address(this), contractNonce);
        c = address(new C());
        require(c == predicted, "Address mismatch");
        contractNonce += 1;

        return c;
    }

    function predictAddress(
        address _deployer,
        uint256 _nonce
    ) public pure returns (address deployed) {
        return LibRLP.computeAddress(_deployer, _nonce);
    }
}

```

The contract `C` is deployed by `Deployer` contract using `new` keyword (this internally uses the `CREATE` opcode).

In our example above, we use `contractNonce` to store the number of deployments, for convenience, since making an RPC call from within a smart contract would require an oracle. Since `contractNonce` is initialized to **1** and updated after each deployment, the predicted address will always match the actual deployed address. Hence, the `deploy()` call will not revert. 

Our example uses `nonce` to store the number of deployments for convenience, as making an RPC call from a smart contract would require an oracle.

```solidity
require(c == predicted, "Address mismatch");
// If this condition is not met, deploy() call will revert 
```

Let’s assume we want to deploy a second contract after a successful first deployment from the deployer’s contract. By that point, `contractNonce` would have incremented to 2 before the second deployment occurs.

 
Here is the result from the `deploy()` call after a second deployment.

![screenshot showing the recorded nonce](https://r2media.rareskills.io/EthereumAddressPrediction/image06.png)

Here is an image showing that the deployed address, from above, matches the address returned from the `predictAddress` call (which calls `computeAddress` from LibRLP).

![deployed contract derivation vs actual](https://r2media.rareskills.io/EthereumAddressPrediction/image07.png)

## How to predict contract address using CREATE2

Create2 was introduced in [EIP-1014](https://eips.ethereum.org/EIPS/eip-1014).

When deploying a contract with the `CREATE2` opcode, its address depends on three components: the deploying contract’s address, a user-supplied salt, and the hash of the contract’s [creation (init) bytecode](https://www.rareskills.io/post/ethereum-contract-creation-code). 

```solidity
Create_contract_address = keccak256(0xff ++ deployer ++ salt ++ keccak256(init_code))
```

Using this relationship, we can precompute the contract’s address, as shown in `getAddress` below:

```solidity
function getAddress(
    bytes memory createCode,
    uint _salt
) public view returns (address) {
    bytes32 hash = keccak256(
        abi.encodePacked(
            bytes1(0xff),
            address(this),
            _salt,
            keccak256(createCode)
        )
    );
    return address(uint160(uint(hash)));
}
```

Where:

- `0xff` is a constant to distinguish `CREATE2` from `CREATE`.
- `salt` is a user-defined value (32 bytes) to ensure uniqueness.
- `keccak256(createCode)` is the hash of the contract’s initialization code.

### Why `CREATE2` prepends with 1 `0xff` byte

`0xff` in the `keccak256` input is the **distinguishing byte** that ensures there's **no collision** between the addresses generated by the `CREATE` and `CREATE2` opcode.

Recall that `CREATE` uses RLP to encode a list of two elements (`[deployer, nonce]`)  for address computation. For context, the deployer’s address is always 20 bytes, while the `nonce` can vary in byte length depending on its value ( [0–8 bytes](https://eips.ethereum.org/EIPS/eip-2681) in practice, but unbounded theoretically). 

Since the **RLP list prefix** is determined by the total [length of the payload](https://ethereum.org/en/developers/docs/data-structures-and-encoding/rlp/), increasing the nonce value can increase the payload length, which in turn affects the prefix. For example, if the payload length is ≤ 55 bytes, the prefix will be in the range: **`0xc0 + payload_length`**.

If the **nonce** is large enough that its RLP-encoded representation exceeds **34 bytes**, it would push the entire `[deployer, nonce]` payload past the 55-byte threshold. Hence, the RLP list prefix will start with a byte in the range `[0xf8, 0xff]`. That said, this case won’t occur in reality, as a 34-byte nonce implies over 17 billion transactions — a number far beyond any plausible usage. 

Moreover, [EIP-2681](https://eips.ethereum.org/EIPS/eip-2681) defines a hard upper bound on nonce as 8 bytes (64 bits), meaning any transaction with a nonce `≥ 2^64-1` is invalid. As a result, the list prefix of `rlp.encode([deployer, nonce])` will always fall within  range `[0xc0, 0xf7]`. 

```solidity
//Here, 0xd6 indicates an RLP list of length 22 bytes. 
rlp.encode([deployer, nonce]) = 0xd6 94 <20-byte deployer> <nonce>
```

So, if `CREATE2` does not prepend with the `0xff` prefix, and simply hashes a raw concatenation like `deployer ++ salt ++ keccak256(init_code)`, then there would be a theoretical (albeit incredibly rare) risk that some chosen values could produce a byte string starting with the same prefix as an RLP-encoded `[deployer, nonce]`. While practically implausible, the domains would not be **provably disjoint**.

By prepending a single `0xff` byte, `CREATE2` ensures that the hashed input always begins with a value (`0xff`) that can never occur at the start of a valid RLP encoding for accounts with realistic nonces. This achieves total domain separation before computing the hash.

### `CREATE2` precomputation example

Now, let us consider an example where we compute a new address from a contract A using the `getAddress` method. Note, this contract has no constructor.

```solidity
contract A {
    address public owner;

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

In the `DeployNewAddr` contract below, the `getAddress` function takes in the **creation bytecode** of contract A and a **salt** value to compute the address at which the contract would be deployed using the `CREATE2` opcode. In this case, the address of `DeployNewAddr` (via `address(this)`) is used in the computation. Hence, the resulting address is dependent on the address of `DeployNewAddr`, the provided salt, and the hash of the creation (init) [bytecode.](https://www.rareskills.io/post/ethereum-contract-creation-code)

```solidity
contract DeployNewAddr {
    function getAddress(
        bytes memory createCode,
        uint _salt
    ) public view returns (address) {
        bytes32 hash = keccak256(
            abi.encodePacked(
                bytes1(0xff),
                address(this),
                _salt,
                keccak256(createCode)
            )
        );
        return address(uint160(uint(hash)));
    }

    function getContractABytecode() public pure returns (bytes memory) {
        bytes memory bytecode = type(A).creationCode;

        return abi.encodePacked(bytecode);
    }
}
```

**Note:** `getAddress()` will return the correct `CREATE2` address **only if the contract that eventually performs the deployment is `DeployNewAddr` itself**. If a different contract performs the deployment using the same bytecode and salt, the resulting address will differ, as the deployer address in the computation (`address(this)`) will not match. When using `getAddress()` outside of the actual deployment context, ensure that the deployer address used in the calculation matches the one that will perform the deployment.

Now, let us consider the case where contract A has a constructor with arguments.

### Handling contracts with constructor arguments in the `getContractABytecode` method

When contracts are deployed (by an EOA or a contract), the EVM executes the contract’s creation code, which consists of the `creationCode`(compiled init bytecode) concatenated with the ABI-encoded constructor arguments. Hence, this behavior is not peculiar to `CREATE2`. 

Recall that, in the previous section, we chose to obtain the `createCode` argument for `getAddress` using a helper function called `getContractABytecode`. Hence, for a contract A with constructor argument(s), this helper function needs to append the argument(s) to the contract’s creation bytecode in the correctly encoded format.

Here is a modified contract A, with one constructor argument.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract A {
    address public owner;

    constructor(address _owner) payable {
        owner = _owner;
    }

    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

If contract A contains a constructor argument as shown above, the `getContractABytecode` function will append the ABI encoding of `_owner` constructor argument as `abi.encodePacked(bytecode, abi.encode(_owner))`. 

If contract A has multiple constructor arguments, as shown below, the deployment bytecode must include all arguments encoded in the correct order, like so: `abi.encodePacked(bytecode, abi.encode(arg1, arg2, ...))`.

```solidity
contract A {
    address public owner;
    address public artMaster;

    constructor(address _owner, address _artMaster) payable {
        owner = _owner;
        artMaster = _artMaster;
    }

    ///*************other logic*************///
}
```

For contract A with constructor arguments `_owner` and `_artMaster`, as shown above, the `getContractABytecode` function will look like this:

```solidity
function getContractAInitByteCode(
    address _owner,
    address _artMaster
) public pure returns (bytes memory) {
    bytes memory bytecode = type(A).creationCode;

    return abi.encodePacked(bytecode, abi.encode(_owner, _artMaster));
}
```

Before we test the `getAddress` method in the `DeployNewAddr`` contract above, let’s look at an alternative `CREATE2` deployment approach that eliminates the need to pass creation bytecode. Instead, it relies on implicit bytecode deployment through Solidity’s native contract instantiation syntax. 

### Deploying (`CREATE2`) contract address without manually passing creation bytecode

This `CREATE2` approach uses Solidity’s built-in `new` keyword along with a `salt` parameter to deploy a contract and return its address. The compiler automatically handles the generation of creation bytecode and the encoding of constructor arguments, so there's no need to pass or construct them manually.

This approach is shown in `DeployNewAddr1` below.

```solidity
contract DeployNewAddr1 {
    // Returns the address of the newly deployed contract
    //DeployNewAddr1, shows a basic deployment with no constructor arguments (A()).
    function deploy(uint _salt) external returns (address x) {
        A Create2NewAddr = new A{salt: bytes32(_salt)};
        return address(Create2NewAddr);
    }
}
```

The `deploy` function in the code above shows the variant of this method, when contract A has no constructor arguments. 

`DeployNewAddr2`, and `DeployNewAddr3`, below, shows how constructor arguments are handled when deploying contract `A` has one and two constructor arguments, respectively.

```solidity
contract DeployNewAddr2 {
    // DeployNewAddr2 includes a single constructor argument _owner,
    // passed to the constructor of contract A.
    // Solidity automatically encodes constructor arguments and appends them to the creation bytecode.
    function deploy(uint _salt, address _owner) external returns (address x) {
        A Create2NewAddr = new A{salt: bytes32(_salt)}(_owner);

        return address(Create2NewAddr);
    }
}

contract DeployNewAddr3 {
    // In DeployNewAddr3, two constructor arguments (msg.sender and _artMaster) are passed to contract A.
    // As in DeployNewAddr2, these arguments are encoded and included in the creation bytecode.
    function deploy(
        uint _salt,
        address _owner,
        address _artMaster
    ) external returns (address x) {
        A Create2NewAddr = new A{salt: bytes32(_salt)}(_owner, _artMaster);

        return address(Create2NewAddr);
    }
}
```

Now, let's run our code (the ones with one constructor argument). 
Let’s see if `deploy` in `DeployNewAddr2` will return the same predicted address as `getAddress` in the `DeployNewAddr` contract. We shall use the salt value of `29` for both methods, with the `getAddress` taking in `bytecode` variable sourced from the `getContractABytecode` function. 

Here is the result obtained from calling both methods:

![create2 deployment demonstration](https://r2media.rareskills.io/EthereumAddressPrediction/image08.png)

From the image above, you can see that the address x returned by the the `deploy` function is the same as the address returned by the `getAddress`  function.

## How to deploy two contract addresses (A and B) that reference each other’s address immutably

Let’s wrap up this tutorial by showing an example of how address prediction can reduce contract deployment costs. 

If we want to deploy two smart contracts ( `A` and `B`), and each contract needs to reference the other’s address. Also, their addresses should never change (i.e., they must be **immutable**).

This setup introduces several challenges that must be addressed:

- Deploying `A` first prevents it from referencing `B`, which does not yet exist.
- Deploying `B` first creates the same issue in reverse—`B` cannot reference `A` before it's deployed.
- Post-deployment, the addresses must be immutable; no setter functions or external updates should be allowed.

One way to solve this problem is to  precompute the addresses of `A` and `B` by using a factory contract address. Then, deploy `A` with `B`’s precomputed address as a constructor argument — and `B` with `A`’s.

Even though this approach is technically correct, it comes with some trade-offs. The factory contract will be deployed and stored on-chain, which increases the overall **bytecode** footprint. Additionally, this approach incurs extra gas costs — both from deploying the factory itself and from executing its logic to deploy the target contracts.

To avoid this overhead, we can use a normal contract deployment and predict the address using the techniques we discussed in this article.

### **Precomputing contract address via Foundry script using the RLP method**

In the steps below, we log the address of our factory account (an EOA), retrieve its current nonce, precompute contract addresses (based on the EOA’s nonce), and deploy them while referencing each other, using the [foundry script](https://book.getfoundry.sh/guides/scripting-with-solidity). 

**Step 1:  Write a script logging the factory (deployer) address, as shown below.**

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "forge-std/Script.sol";
import {A, B} from "../src/DeployAddr.sol";

contract DeployAddrScript is Script {
    A public a;

    function run() public {
        uint256 pk = vm.envUint("PRIV_KEY");
        address dep = vm.addr(pk);
        //WARNING: With vm.envUint, the private key is loaded in cleartext into memory
        //NEVER use this pattern in production or with private keys managing real funds.
        //Assume any key kept in .env will eventually be stolen

        console.log("This is the deployer's address:", dep);

        vm.startBroadcast(pk);
        new A(address(0));

        vm.stopBroadcast();
    }
}
```

Terminal returns:

```solidity
$ forge script script/DeployAddr.s.sol --rpc-url http://localhost:8545
[⠒] Compiling...
No files changed, compilation skipped
Script ran successfully.

== Logs ==
  This is the deployer's address: 0x8768C6FB71815b2e8Ab6dD31b67a926781aC8f1A
```

**Step 2: Compute the address of contracts A and B.**

Now that we have our deployer’s address from our private key, we can deterministically generate the address using the command `cast compute-address <address> --nonce <value>`. 

See the result for nonce 0 and nonce 1 below for the deployer’s address `0x8768C6FB71815b2e8Ab6dD31b67a926781aC8f1A`:

```solidity
$ cast compute-address 0x8768C6FB71815b2e8Ab6dD31b67a926781aC8f1A --nonce 0
Computed Address: 0x9b4393C60f2408de53F04d93aD178ffBAF25b202

user@DESKTOP-QOJ9UFF MINGW64 ~/Desktop/testFile (master)
$ cast compute-address 0x8768C6FB71815b2e8Ab6dD31b67a926781aC8f1A --nonce 1
Computed Address: 0x20cf99233e5B16Fba6B0E7bA70768d6EDe75789D

```

**Note:** Using the wrong **nonce** will result in an incorrect address. For example, in the script above, once `new A(address(0))` is deployed (using an EOA), the deployer's nonce increments from `0` to `1`.

Computing an address using nonce `0` after that deployment will lead to a mismatch in the contract's address.

Alternatively, we can determine the addresses A and B using `vm.getNonce` cheatcode and `computeAddress`, as shown below.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "forge-std/Script.sol";
import {A, B} from "../src/DeployAddr.sol";
import {LibRLP} from "lib/LibRLP.sol";

contract DeployAddrScript is Script {
    A public a;

    //B public b;

    function run() public {
        uint256 pk = vm.envUint("PRIV_KEY");
        address dep = vm.addr(pk);

        console.log("This is the deployer's address:", dep);

        vm.startBroadcast(pk);

        //nonce = 0,
        new A(address(0));
        //Deploys a new instance of contract A, passing in address(0) as a constructor argument.
        // after this, nonce = 1.

        // compute the current nonce for the address
        uint256 currentNonce = vm.getNonce(dep);

        console.log("This is the current nonce: %s", currentNonce);

        address predicted_a = LibRLP.computeAddress(dep, currentNonce);
        address predicted_b = LibRLP.computeAddress(dep, currentNonce + 1);

        console.log("predicted_a: %s", predicted_a);
        console.log("predicted_b: %s", predicted_b);

        vm.stopBroadcast();
    }
}
```

Here is the terminal result after running the script:

```solidity
Script ran successfully.

== Logs ==
  This is the deployer's address: 0x8768C6FB71815b2e8Ab6dD31b67a926781aC8f1A
  This is the current nonce: 1
  
  predicted_a: 0x20cf99233e5B16Fba6B0E7bA70768d6EDe75789D
  predicted_b: 0xca3fF2a864026daC337312142Aa71D57c7D8Dde3
```

**Step 3: Deploy the contracts with their corresponding constructor arguments (i.e., the precomputed addresses).**

Now, let us deploy the contracts (`A` and `B`) and compare the results with `predicted_a` and `predicted_b`.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import {Script, console} from "forge-std/Script.sol";
import {A, B} from "../src/DeployAddr.sol";
import {LibRLP} from "lib/LibRLP.sol";

contract DeployAddrScript is Script {
    A public a;
    B public b;

    function run() public {
        uint256 pk = vm.envUint("PRIV_KEY");
        address dep = vm.addr(pk);

        console.log("This is the deployer's address:", dep);

        vm.startBroadcast(pk);

        // compute the current nonce for the address

        uint256 currentNonce = vm.getNonce(dep);

        console.log("This is the current nonce: %s", currentNonce);

        address predicted_a = LibRLP.computeAddress(dep, currentNonce);
        address predicted_b = LibRLP.computeAddress(dep, currentNonce + 1);

        A a = new A(predicted_b);
        B b = new B(predicted_a);

        console.log("address(a): %s", address(a));
        console.log("predicted_a: %s", predicted_a);
        console.log("address(b): %s", address(b));
        console.log("predicted_b: %s", predicted_b);

        vm.stopBroadcast();
    }
}
```

 

Here is the terminal result:

```solidity
Script ran successfully.

== Logs ==
  This is the deployer's address: 0x8768C6FB71815b2e8Ab6dD31b67a926781aC8f1A
  This is the current nonce: 1
  
  address(a): 0x20cf99233e5B16Fba6B0E7bA70768d6EDe75789D
  predicted_a: 0x20cf99233e5B16Fba6B0E7bA70768d6EDe75789D
  address(b): 0xca3fF2a864026daC337312142Aa71D57c7D8Dde3
  predicted_b: 0xca3fF2a864026daC337312142Aa71D57c7D8Dde3
```

We can see in the terminal result that the deployed address, a and b, corresponds with the predicted addresses predicted_a and predicted_b, respectively.

## Conclusion

In this article, we explored how Ethereum contract addresses are predicted across different deployment methods. For contracts deployed using the `CREATE` opcode, we showed that the resulting address depends only on the deployer’s address and nonce — constructor arguments — bytecode plays no role. For `CREATE2`, we explained how address prediction incorporates a salt and the keccak256 hash of the full creation bytecode, including constructor arguments. Finally, we described how to efficiently precompute and deploy two interdependent contracts off-chain using Foundry scripts and `computeAddress`.

## EIPs referenced in this guide

[EIP-161](https://eips.ethereum.org/EIPS/eip-161): Defines Account creation transactions, introduces the concept of "empty accounts," nonce handling, and rules for their cleanup.
[EIP-1014](https://eips.ethereum.org/EIPS/eip-1014): Introduces the `CREATE2` opcode.
[EIP-2681](https://eips.ethereum.org/EIPS/eip-2681): Defines the Limit of account nonce to be between `0` and `2^64-1`.
