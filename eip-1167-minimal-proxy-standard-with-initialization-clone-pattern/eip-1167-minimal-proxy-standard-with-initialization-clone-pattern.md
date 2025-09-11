# EIP-1167: Minimal Proxy Standard with Initialization (Clone pattern)

![clones](https://static.wixstatic.com/media/935a00_ab231f835070402e9b8e73bcc5037668~mv2.jpg/v1/fill/w_740,h_475,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_ab231f835070402e9b8e73bcc5037668~mv2.jpg)
Image from https://pixabay.com/photos/stormtrooper-star-wars-lego-storm-2899993/

## Introduction

EIP-1167, which is also referred to as the minimal proxy contract, is a commonly used [solidity](https://www.rareskills.io/solidity-bootcamp) pattern for cheaply creating proxy clones.

If a use case requires deploying an identical contract (or very similar contract) repeatedly, this is a more gas efficient way to do it.

For example, [gnosis safe](https://safe.global/) uses the clone pattern when creating a new safe. When you interact with gnosis safe, you are actually interacting with a clone of it.

A clone contract is like a proxy that cannot be upgraded. Because proxies are very small relative to the implementation contract, they are cheap to deploy.

Like the proxy pattern, clones delegate all calls to the implementation contract but keep state in their own storage.

Unlike the regular proxy pattern, several clones can point to the same implementation contract. Clones cannot be upgraded.

The address of the implementation contract is stored in the bytecode. This saves gas compared to storage and prevents the clone from pointing to another implementation.

This design makes it considerably cheaper to deploy since the bytecode of a clone proxy is typically much smaller than the bytecode of an implementation contract. In fact, the EIP-1167 is only 55 bytes large (45 bytes for the runtime), including the init code. However, calls during execution will cost more, because there is always an added delegatecall.

This article will describe both the EIP and the initializer function used to initialize the equivalent of the constructor parameters.

### Authorship

This article was co-written by Jesse Raymond ([LinkedIn](https://www.linkedin.com/in/jesse-raymond4/), [Twitter](https://twitter.com/Jesserc_)) as part of the RareSkills Technical Writing Program.

## How EIP-1167 works

As a typical proxy, it receives transaction data via a call, forwards this data to the implementation smart contract, gets the result of the external call, and returns the result if the external call was successful or reverts if there was an error.

## The bytecode of the minimal proxy

The minimal proxy contract has a concise bytecode of only 55 bytes. This bytecode comprises of the

- init code
- the runtime code which includes instructions to receive transaction calldata
- the 20-byte implementation contract address
- and commands to execute a delegate call and
- return the result, or trigger a revert if an error occurs.

Here is the bytecode of the minimal proxy:

```evm-bytecode
3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe5af43d82803e903d91602b57fd5bf3
```

The dummy address: `0xbebebebebebebebebebebebebebebebebebebebe` is replaced with the implementation contract address.

Let's break it down.

![bytecode of the clone contract with regions of the bytecode highlighted](https://static.wixstatic.com/media/935a00_bbb238c84bbc4100aaf93e61ef956a86~mv2.png/v1/fill/w_740,h_128,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_bbb238c84bbc4100aaf93e61ef956a86~mv2.png)

### The init code section

The first 10 bytes of the bytecode contain the init code which runs once and is used for deploying the minimal proxy.

To learn more about smart contract creation and deployments, see our article on [Ethereum smart contract creation code](https://www.rareskills.io/post/ethereum-contract-creation-code).

Here are the following commands that are carried out in the EVM.

```evm-bytecode
// copy the runtime bytecode of the minimal proxy
// starting from offset 10, and save it to the blockchain

[00] RETURNDATASIZE
[01] PUSH1    2d
[03] DUP1

//push 10 - offset to copy runtime code from
[04] PUSH1    0a
[06] RETURNDATASIZE

// copy the runtime code and save it to the blockchain
[07] CODECOPY
[08] DUP2
[09] RETURN
```

### Copying calldata

The init code deploys the contract and saves the runtime bytecode on-chain, starting from offset 10 (the copy calldata section) to the end of the bytecode.

After a minimal proxy has been deployed and a call is sent to it, it copies the transaction calldata to memory, pushes the 20 bytes address of the implementation contract, and performs a delegatecall on the implementation contract.

This calldata copying is done with the following opcodes.

```evm-bytecode
//copy the transaction calldata to memory
[0a] CALLDATASIZE
[0b] RETURNDATASIZE // this is a hack to push 0 onto the stack with less gas than doing PUSH 0
[0c] RETURNDATASIZE
[0d] CALLDATACOPY

[0e] RETURNDATASIZE
[0f] RETURNDATASIZE
[10] RETURNDATASIZE
[11] CALLDATASIZE
[12] RETURNDATASIZE

//pushes the 20 bytes address of the implementation contract
[13] PUSH20
```

### The implementation contract address

After the transaction calldata is copied to memory, the stack is prepared to make a delegatecall and the 20 bytes address of the implementation contract is pushed to the top of the stack. In the previous section, we can see that it ends with a PUSH20. What follows next is the address of the implementation contract.

```evm-bytecode
//push the address of the implementation contract to the stack. The address here is just a dummy address
[13] PUSH20 bebebebebebebebebebebebebebebebebebebebe
```

### The delegatecall section

After copying the transaction calldata to memory and getting the implementation contract address at the top of the stack, the minimal proxy is ready to perform a delegatecall to the implementation contract.

If you need a refresher on how delegatecall works, read our tutorial on [delegatecall](https://docs.soliditylang.org/en/v0.8.6/introduction-to-smart-contracts.html?highlight=delegatecall#delegatecall-callcode-and-libraries).

After the delegate call is performed, the minimal proxy returns the result of the call if it was successful or reverts if an error had occurred.

The delegatecall section has the following opcodes.

```evm-bytecode
//perform a delegate call on the implementation contract, and forward all available gas
[28] GAS
[29] DELEGATECALL

//copy the return data of the call to memory
[2a] RETURNDATASIZE
[2b] DUP3
[2c] DUP1
[2d] RETURNDATACOPY

// set up the stack for the conditional jump
[2e] SWAP1
[2f] RETURNDATASIZE
[30] SWAP2
[31] PUSH1    2b

//jump to line 35 and return the result of the call if it was successful, else revert on line 34
[33] JUMPI
[34] REVERT
[35] JUMPDEST
[36] RETURN
```

This is an overview of the EIP-1167 and how it works.

Imagine the implementation contract is an ERC20 token. In that case, the clone will behave exactly like an ERC20 token.

## EIP-1167 smart contract implementation with initialization

There are some situations where we want to parameterize the creation of a clone. For example, if we were cloning an ERC20 token, every clone would have the same `totalSupply`, which might not be desirable.

To be able to configure this parameter, the "clone with initialization pattern" can be used.

Let's see how the EIP-1167 can be used to create proxy clones with an initialization function. This follows a simple sequence of steps:

1. Create an implementation contract
2. Clone the contract with the EIP-1167 standard
3. Deploy the clone and call the initialization function, which can only be called once.
This restriction to only call once is necessary, or someone might alter the critical parameter we set after deployment, such as changing the total supply.

Let's go through these steps with an example below.

The implementation contract to be cloned

```solidity
contractImplementationContract{
    boolprivate isInitialized;      //initializer function that will be called once, during
    deployment.

    functioninitializer() external {
        require(!isInitialized);
        isInitialized =true;
    }          // rest of the implementation functions go here
}
```

We use this code to deploy the clone

```solidity
contract MinimalProxyFactory {

    address[] public proxies;

    function deployClone(address _implementationContract) external returns (address) {
        // convert the address to 20 bytes
        bytes20 implementationContractInBytes = bytes20(_implementationContract);
        //address to assign a cloned proxy
        address proxy;


        // as stated earlier, the minimal proxy has this bytecode
        // <3d602d80600a3d3981f3363d3d373d3d3d363d73><address of implementation contract><5af43d82803e903d91602b57fd5bf3>

        // <3d602d80600a3d3981f3> == creation code which copies runtime code into memory and deploys it

        // <363d3d373d3d3d363d73> <address of implementation contract> <5af43d82803e903d91602b57fd5bf3> == runtime code that makes a delegatecall to the implementation contract


        assembly {
            /*
            reads the 32 bytes of memory starting at the pointer stored in 0x40
            In solidity, the 0x40 slot in memory is special: it contains the "free memory pointer"
            which points to the end of the currently allocated memory.
            */
            let clone := mload(0x40)
            // store 32 bytes to memory starting at "clone"
            mstore(
                clone,
                0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000
            )

            /*
              |              20 bytes                |
            0x3d602d80600a3d3981f3363d3d373d3d3d363d73000000000000000000000000
                                                      ^
                                                      pointer
            */
            // store 32 bytes to memory starting at "clone" + 20 bytes
            // 0x14 = 20
            mstore(add(clone, 0x14), implementationContractInBytes)

            /*
              |               20 bytes               |                 20 bytes              |
            0x3d602d80600a3d3981f3363d3d373d3d3d363d73bebebebebebebebebebebebebebebebebebebebe
                                                                                              ^
                                                                                              pointer
            */
            // store 32 bytes to memory starting at "clone" + 40 bytes
            // 0x28 = 40
            mstore(
                add(clone, 0x28),
                0x5af43d82803e903d91602b57fd5bf30000000000000000000000000000000000
            )

            /*
            |                 20 bytes                  |          20 bytes          |           15 bytes          |
            0x3d602d80600a3d3981f3363d3d373d3d3d363d73b<implementationContractInBytes>5af43d82803e903d91602b57fd5bf3 == 45 bytes in total
            */


            // create a new contract
            // send 0 Ether
            // code starts at the pointer stored in "clone"
            // code size == 0x37 (55 bytes)
            proxy := create(0, clone, 0x37)
        }

        // Call initialization
        ImplementationContract(proxy).initializer();
        proxies.push(proxy);
        return proxy;
    }
}
```

With the `MinimalProxyFactory` contract, an infinite amount of EIP-1167 clones can be deployed, but for this example, we will be deploying the implementation contract we have above.

Here's a simple hardhat script that deploys the contracts and interacts with a deployed clone.

```javascript
const hre = require("hardhat");

async function main() {
  const ImplementationContract = await hre.ethers.getContractFactory(
    "ImplementationContract"
  );
  // deploy the implementation contract
  const implementationContract = await ImplementationContract.deploy();
  await implementationContract.deployed();

  console.log("Implementation contract ", implementationContract.address);

  const MinimalProxyFactory = await hre.ethers.getContractFactory(
    "MinimalProxyFactory"
  );
  // deploy the minimal factory contract
  const minimalProxyFactory = await MinimalProxyFactory.deploy();
  await minimalProxyFactory.deployed();

  console.log("Minimal proxy factory contract ", minimalProxyFactory.address);

  // call the deploy clone function on the minimal factory contract and pass parameters
  const deployCloneContract = await minimalProxyFactory.deployClone(
    implementationContract.address
  );
  deployCloneContract.wait();

  // get deployed proxy address
  const ProxyAddress = await minimalProxyFactory.proxies(0);
  console.log("Proxy contract ", ProxyAddress);

  // load the clone
  const proxy = await hre.ethers.getContractAt(
    "ImplementationContract",
    ProxyAddress
  );

  console.log("Proxy is initialized == ", await proxy.isInitialized()); // get initialized boolean == true
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

We have now deployed our contracts on the goerli network and here are the transaction details for the 3 contracts.
- [The implementation contract](https://goerli.etherscan.io/address/0x4D11C446473105a02B5C1ab9EBE9b03F33902A29#code)
- [The minimal proxy factory contract](https://goerli.etherscan.io/address/0x8c7a30C5e64a088e1b03E2012E9ff42398AD0DA8#code)
- [The minimal proxy contract](https://goerli.etherscan.io/address/0x3348f2aee62a0ddb164c711b5937e4001c17080e#code)

Note that Etherscan recognizes the proxy contract is not just another smart contract, but delegates call to the implementation contract.

For the sake of convenience, our code keeps a list of clones deployed, but this is not a necessary feature.

## Conclusion

The EIP-1167 minimal proxy standard is an efficient way to deploy contracts that mirror the implementation of another one. The initializer pattern allows us to deploy the clone as if it has a constructor that takes arguments.

The cost to this pattern is that every execution has a delegate call overhead.

### Learn More

See our advanced [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) to see our extensive curriculum.

*Originally Published Feb 21, 2023*
