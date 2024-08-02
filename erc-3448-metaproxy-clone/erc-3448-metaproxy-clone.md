# EIP-3448 MetaProxy Standard: Minimal Proxy with support for immutable metadata

The minimal proxy standard allows us to parameterize the creation of the clone, but this requires an extra initialization transaction. It is possible to bypass this step entirely and parameterize the value we care about in the bytecode of the proxy, instead of using storage.

The MetaProxy standard is also a minimal bytecode implementation for creating smart contract clones with an added unique immutable metadata for each of the clones.

This metadata can be anything, from a string to a number, and can have an arbitrary length. However, the intended use is to serve as function arguments to parameterize the behavior of the implementation contracts.

Since the bytecode of this standard is known, it can be used by users and third-party tools, such as Etherscan to figure out that a clone will always redirect to a particular implementation contract address along with the appended metadata.

Let’s take a look at the bytecode of the MetaProxy without the metadata.

```
600b380380600b3d393df3363d3d373d3d3d3d60368038038091363936013d73bebebebebebebebebebebebebebebebebebebebe5af43d3d93803e603457fd5bf3
```

The length of the Metaproxy bytecode is 65 bytes, consisting of an 11 bytes init code and 54 bytes runtime code.

Although the bytecode of the MetaProxy contract is similar to the Minimal Proxy standard, some parts of the bytecode are different, for example, the green section of the bytecode (below) has some additional opcode commands, which we will explain later.

![ERC-3448 metaproxy bytecode with relevant sections highlighted](https://static.wixstatic.com/media/935a00_99e0747e4b094c79b9e2893abc82fadb~mv2.png/v1/fill/w_666,h_126,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_99e0747e4b094c79b9e2893abc82fadb~mv2.png)

The dummy address `0xbebebebebebebebebebebebebebebebebebebebe` is replaced with the implementation contract address after deployment.

### Authorship

This article was co-written by Jesse Raymond ([LinkedIn](https://www.linkedin.com/in/jesse-raymond4/), [Twitter](https://twitter.com/Jesserc_)) as part of the RareSkills Technical Writing Program.

## Creating an ERC20 contract with the MetaProxy standard

In this section, we will be creating a MetaProxy clone of an ERC20 contract. Let’s delve into how this can be done and visualize how the metadata is added to the clone.

To implement the ERC20 contract, we will inherit the OpenZeppelin `ERC20Upgradeable` contract, which has an “ERC20_init” function used for initializing ERC20 state variables instead of a constructor, which cannot be used with proxy patterns like the one we are building here.

This is because constructors are called on contract deployments, and if we were to follow this method, the state variables like `name` and `symbol` of the ERC20 standard would not be initialized in the ERC20 MetaProxy clone bytecode, because the constructor would be setting the storage of the implementation contract and not the clone.

However, we won’t be using the initialization function because we can simply get the `name`, `symbol` and `totalSupply` of the ERC20 MetaProxy clone from its bytecode after adding them as metadata.

### The ERC20 implementation contract

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts-upgradeable/token/ERC20/ERC20Upgradeable.sol";

contract ERC20Implementation is ERC20Upgradeable {
    // get ERC20 name from the metadata
    function name()
        public
        view
        virtual
        override
        returns (string memory name__)
    {
        (name__, , ) = getMetadata();
    }

    // get ERC20 symbol from the metadata
    function symbol()
        public
        view
        virtual
        override
        returns (string memory symbol__)
    {
        (, symbol__, ) = getMetadata();
    }

    // get ERC20 total supply from the metadata
    function totalSupply()
        public
        view
        virtual
        override
        returns (uint256 totalSupply_)
    {
        (, , totalSupply_) = getMetadata();
    }

    // mint function
    function mint(uint amount) public {
        _mint(msg.sender, amount * 10 ** 18);
    }

    /// returns the decoded metadata of this (ERC20 MetaProxy) contract.
    function getMetadata()
        public
        pure
        returns (
            string memory name__,
            string memory symbol__,
            uint256 totalSupply__
        )
    {
        bytes memory data;
        assembly {
            let posOfMetadataSize := sub(calldatasize(), 32)
            let size := calldataload(posOfMetadataSize)
            let dataPtr := sub(posOfMetadataSize, size)
            data := mload(64)
            mstore(64, add(data, add(size, 32)))
            mstore(data, size)
            let memPtr := add(data, 32)
            calldatacopy(memPtr, dataPtr, size)
        }

        //return the decoded the metadata
        return abi.decode(data, (string, string, uint256));
    }
}
```


### Getting the metadata

The `getMetadata` function is used in the implementation to return the clone’s metadata. Since the MetaProxy always loads its metadata whenever its functions are called (this is the design of the standard, which we will explain later in this article), the `getMetadata` function is used to extract the metadata from the call and return it as a tuple in our implementation.

It is also being used in the ERC20 `name`, `symbol` and `totalSupply` functions to get a specific part of the metadata, either a string for `name` and `symbol` or a uint256 for the `totalSupply`.

We have derived this function from the example implementation [here](https://github.com/ethereum/EIPs/blob/5d51b43cf4f36a7445e546ec809a6f0d888bb191/assets/eip-3448/MetaProxyTest.sol#L56) and modified to suit our purposes for the ERC20 contract.

### The Factory contract

The original EIP also has a [link](https://github.com/ethereum/EIPs/blob/0b7432e2dc085a4e1fff7c9de993c74b7e9e9abf/assets/eip-3448/MetaProxyFactory.sol#L46) to the implementation of the `MetaProxyFactory`, which we import and inherit here.

The `MetaProxyFactory` contains the code for creating new MetaProxy clones.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;
import "./ERC20Implementation.sol";
import "./MetaProxyFactory.sol";

contract ERC20MetaProxyFactory is MetaProxyFactory {
    address[] public proxyAddresses;

    function createClone(
        string memory _name,
        string memory _symbol,
        uint256 _initialSupply
    ) public returns (address) {
        // Encode the ERC20 constructor arguments
        bytes memory metadata = abi.encode(_name, _symbol, _initialSupply);

        // Create the proxy
        address proxyAddress = _metaProxyFromBytes(
            address(new ERC20Implementation()),
            metadata
        );

        proxyAddresses.push(proxyAddress);
        return proxyAddress;
    }
}
```

### Creating a clone - explaining the factory contract

The `ERC20MetaProxyFactory` is our factory contract for creating new clones. We use the `_metaProxyFromBytes` function which is inherited from the `MetaProxyFactory`, to deploy new clones.

The `_metaProxyFromBytes` function takes in two arguments, which are; 1. the address of the implementation contract (this is why we use the new keyword to deploy an “ERC20Implementation” contract first). 2. the metadata.

Since the bytecode of smart contracts is represented in hexadecimal in this code, the metadata must be [abi encoded](https://www.rareskills.io/post/abi-encoding) before it can be appended to the clone's bytecode.

This is why we encode the `createClone` function arguments before passing them as metadata to the `_metaProxyFromBytes` function which creates the new clone and returns the address.

This is the function signature of the `_metaProxyFromBytes` function.

```solidity
function _metaProxyFromBytes (address targetContract, bytes memory metadata) internal returns (address) {
     // code that deploys new clones here
 }
```

### Deploying the clone

Here’s a hardhat script that deploys the contracts and interacts with a deployed clone on the sepolia network:

```javascript=
const hre = require("hardhat");

async function main() {
  const ERC20ProxyFactory = await hre.ethers.getContractFactory(
    "ERC20MetaProxyFactory"
  );

  const erc20ProxyFactory = await ERC20ProxyFactory.deploy();
  // deploy the erc20 proxy factory contract
  await erc20ProxyFactory.deployed();

  console.log(
    `ERC20 proxy factory contract deployed to ${erc20ProxyFactory.address}`
  );

  // create clone
  const tx1 = await erc20ProxyFactory.createClone(
    "Meta Token V1",
    "MTV1",
    "150000000000000000000000" //150,000 initial supply * 10^18 decimals
  );
  await tx1.wait();

  const proxyCloneAddress = await erc20ProxyFactory.proxyAddresses(0);
  console.log("Proxy clone deployed to", proxyCloneAddress);

  // load the clone
  const proxyClone = await hre.ethers.getContractAt(
    "ERC20Implementation",
    proxyCloneAddress
  );

  // retrieve the metadata
  const metadata = await proxyClone.getMetadata();
  console.log("metadata for clone: ", metadata);
  //retrieve the "name" string from the metadata
  const name = await proxyClone.name();
  console.log("ERC20 name of clone from metadata: ", name);

  const tx2 = await proxyClone.mint(150_000);
  tx2.wait();
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```


After running this script, the console displayed the following output:

```bash
ERC20 proxy factory contract deployed to 0xd45f2c555ba30aCb89EB0a3fff6a4416f8cC06e2
Proxy clone deployed to 0x5170672424194899F52B29E60e85C1632F0C732e
metadata for clone:  [
  'MetaProxy Token',
  'MPRXT',
  BigNumber { value: "150000000000000000000000" },
  name__: 'MetaProxy Token',
  symbol__: 'MPRXT',
  totalSupply__: BigNumber { value: "150000000000000000000000" }
]
ERC20 name of clone from metadata:  MetaProxy Token
```

We have also deployed our contracts on the sepolia network and here are the details for the 3 contracts.

1.  [The MetaProxy factory contract](https://sepolia.etherscan.io/address/0xd45f2c555ba30aCb89EB0a3fff6a4416f8cC06e2#code)
    
2.  [The ERC20 implementation contract](https://sepolia.etherscan.io/address/0x20d9cd49f7d235637bb48c88f1f77308f85673ac#code)
    
3.  [The ERC20 MetaProxy contract](https://sepolia.etherscan.io/address/0x5170672424194899F52B29E60e85C1632F0C732e#readProxyContract)

Notice the "read" and "write as proxy" for the ERC20 MetaProxy contract, this means that Etherscan recognizes the proxy contract is not just another smart contract, but a proxy contract.

For the sake of convenience, our code keeps a list of clones deployed so we can easily access them in our hardhat environment, but this is optional.

### Handling Reverts

As stated in the introduction, if an error occurs when the proxy clone redirects a call to the implementation contract, the revert payload is returned to the clone and displayed to users.

Let’s put this to the test and see if it works as expected.

In our previous ERC20 contract example, we will attempt to call the “transferFrom” function with no allowance to see if the transaction is successful or if the error is returned to us.

We do this with this hardhat script.

```javascript
try {
     await proxyClone.transferFrom(
     proxyCloneAddress,
     erc20ProxyFactory.address,2000000000);
} catch (error) {
   console.error(error);
}
```

And boom! we got an error.

```
Error: VM Exception while processing transaction: reverted with reason 
string 'ERC20: insufficient allowance'
```

This means that the revert and the reason for the reverts are sent back to the clone nicely!

## Explaining the bytecode of the deployed ERC20 clone

Remember we said earlier that the metadata of the clone is appended to the end of the clone. In this section, we will explain the deployed bytecode of the MetaProxy clone.

Note that the bytecode of all clones follows the minimal bytecode of the MetaProxy standard, except that each clone has its metadata at the end of its bytecode.

Let us look at the bytecode of the ERC20 clone.

```
0x363d3d373d3d3d3d60368038038091363936013d731bf70065f6b4e424b7b642b3a76a5e01f208e3fc5af43d3d93803e603457fd5bf3000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000174876e800000000000000000000000000000000000000000000000000000000000000000a50726f7879546f6b656e00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000650546f6b656e000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e0
```

That’s a lot of zeros right there! Our bytecode is 310 bytes long.

Let’s break this down further.

```
<=== the runtime bytecode of the MetaProxy standard ===>
0x363d3d373d3d3d3d60368038038091363936013d731bf70065f6b4e424b7b642b3a76a5e01f208e3fc5af43d3d93803e603457fd5bf3
<===>
<=== the abi encoded metadata ===>
000000000000000000000000000000000000000000000000000000000000006000000000000000000000000000000000000000000000000000000000000000a0000000000000000000000000000000000000000000000000000000174876e800000000000000000000000000000000000000000000000000000000000000000a50726f7879546f6b656e00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000650546f6b656e000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000e0
<===>
```

The encoded metadata is comprised of the offsets in memory where the encoded values are stored, the length of the encoded string values, the values, and the next available memory (the free memory pointer). Here’s a detailed breakdown of the metadata.

Following the ABI specification, the first three sets of 32-byte words represent either the values of the data we are encoding or the pointer to their location if they have a dynamic type. We have two strings, and one unsigned integer, representing the name, symbol, and total supply respectively. Because name and symbol are dynamic, their slots contain pointers, whereas the total supply is simply stored in its slot.

```
// memory[0x00 - 0x20] 0000000000000000000000000000000000000000000000000000000000000060 // memory offset for name string
// memory[0x20 - 0x40] 00000000000000000000000000000000000000000000000000000000000000a0 // memory offset for symbol string
// memory[0x40 - 0x60] 000000000000000000000000000000000000000000000000000000174876e800 // the encoded total supply (uint256)
// memory[0x60 - 0x80] 000000000000000000000000000000000000000000000000000000000000000a // the length of the name string (0x0a == 10)
// memory[0x80 - 0xa0] 50726f7879546f6b656e00000000000000000000000000000000000000000000 // the encoded name string
// memory[0xa0 - 0xc0] 0000000000000000000000000000000000000000000000000000000000000006 // the length of the symbol string (6)
// memory[0xc0 - 0xe0] 50546f6b656e0000000000000000000000000000000000000000000000000000 // the encoded symbol string
// memory[0xe0       ] 00000000000000000000000000000000000000000000000000000000000000e0 // the length of the metadata (0xe0 == 224)

```

As stated earlier, the runtime code is 54 bytes. If we split the bytecode of the ERC20 clone into two, taking out the first 54 bytecode of the runtime code, we are left with the abi encoded metadata, which is 224 bytes, and the length of the metadata added to the end of the code, which is 32 bytes.

According to the [standard](https://eips.ethereum.org/EIPS/eip-3448)

> …everything after the MetaProxy bytecode can be arbitrary metadata and the last 32 bytes (one word) of the bytecode must indicate the length of the metadata in bytes.

In our case, the metadata is 224 bytes long, and its length is stored in the final 32 bytes (0x000…000e0).

It may seem odd to store the length of the metadata at the end, as ABI encoding typically stores length before the data begins, but in this case, it makes it easy for the implementation contract to parse the extra metadata with the following code from earlier.

```solidity
let posOfMetadataSize := sub(calldatasize(), 32)
```

If we decode the metadata [here](https://adibas03.github.io/online-ethereum-abi-encoder-decoder/#/decode), we get the initialization data of the clone.

![decoding proxy token](https://static.wixstatic.com/media/935a00_d037c1d5e5c34242878e5962db23ede4~mv2.png/v1/fill/w_666,h_306,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_d037c1d5e5c34242878e5962db23ede4~mv2.png)

Let’s walk through the mnemonics of the bytecode.

```
<=== start of the runtime bytecode ===>

// Note that RETURNDATASIZE is used in some parts of the bytecode to push zero to the stack.
// This is because RETURNDATASIZE (2 gas) costs less gas than a PUSH1 0 (3 gas).

// copy transaction calldata
[00]	CALLDATASIZE	
[01]	RETURNDATASIZE	
[02]	RETURNDATASIZE	
[03]	CALLDATACOPY	

// prepare the stack for a delegate call
[04]	RETURNDATASIZE	
[05]	RETURNDATASIZE	
[06]	RETURNDATASIZE	
[07]	RETURNDATASIZE	
[08]	PUSH1	36		// 0x36 == 54, this is the length of the runtime code
[0a]	DUP1	
[0b]	CODESIZE		// get the length of the clone's bytecode + the metadata, which is 310 bytes
[0c]	SUB    			// subtract the runtime code from the bytecode, to get the metadata (the remaining 256 bytes). this is used in the delegatecall
[0d]	DUP1	
[0e]	SWAP2	
[0f]	CALLDATASIZE	
[10]	CODECOPY               // copy the metadata to memory and forward it to the implementation contract during the delegatecall.
[11]	CALLDATASIZE	
[12]	ADD	
[13]	RETURNDATASIZE	

// push the address of the implementation contract to the stack and perform the delegatecall
[14]	PUSH20	1bf70065f6b4e424b7b642b3a76a5e01f208e3fc  
[29]	GAS	
[2a]	DELEGATECALL	

// copy the return data (the result of the call) to memory and set up the stack for a conditional jump
[2b]	RETURNDATASIZE	
[2c]	RETURNDATASIZE	
[2d]	SWAP4	
[2e]	DUP1	
[2f]	RETURNDATACOPY	
[30]	PUSH1	34

//jump to line 34 and return the result of the call if it was successful, else revert on line 33
[32]	JUMPI	
[33]	REVERT	
[34]	JUMPDEST	
[35]	RETURN

<<=== the metadata starts from here ===>>
```

This is a high-level description of how the bytecode of the clone works. In summary, it copies calldata sent to it in a transaction and makes a delegatecall to the implementation contract with that calldata while forwarding the metadata in the delegatecall.

Note that since the metadata is also forwarded in all calls, it’ll be sent along with some functions that do not need it, like the ERC20 `balanceOf()` function, which has no relation with the encoded metadata.

## Conclusion

The EIP-3448 MetaProxy standard can be viewed as an extension to the EIP-1167 Minimal Proxy Standard, allowing immutable metadata to be attached to each clone’s runtime bytecode.

The MetaProxy standard allows users to parameterize the value they care about in the clone’s bytecode rather than using storage, which reduces gas costs.

Furthermore, third-party tools such as Etherscan can use the known bytecode of the standard to determine that a clone will always redirect to a specific implementation contract address in a specific manner and query the metadata attached to a clone.

### Learn More

This material is part of our Advanced [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp). See our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) for all our offerings.

*Originally Published March 3, 2023*