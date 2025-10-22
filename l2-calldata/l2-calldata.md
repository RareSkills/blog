# Layer 2 Calldata Gas Optimization

**Update for mid 2024**: As of the Dencun upgrade, calldata optimization doesn't have as much of an impact since the transactions on most L2s are stored on blobs, instead of calldata. We keep this article for historical purposes.

When developing applications on an L2, the majority of gas costs come from calldata. Therefore, gas optimization for L2 emphasizes minimizing that cost.

This article explores how calldata optimization works, provide some examples, and discuss chain-specific techniques.

## Prerequisites

-   The reader should be familiar with Solidity and Ethereum Virtual Machine (EVM).
-   The reader should at least know some simple [gas optimization](https://www.rareskills.io/post/gas-optimization) techniques.
-   The reader should know what ABI encoding/decoding, this [video on ABI encoding](https://www.rareskills.io/post/abi-encoding) is a good starting point to learn.

## Authorship

This article was written by Rati Montreewat ([LinkedIn](https://www.linkedin.com/in/rati-montreewat/), [Twitter](https://twitter.com/RATi_MOn)), a blockchain engineer and an author of [Solid Grinder](https://github.com/Ratimon/solid-grinder), an L2 calldata optimization tool, and an alumni of the RareSkills [Solidity Bootcamp](https://hackmd.io/6dXGm0IxQJWongosS25O2w?view).

## The Cost of Calldata

Ethereum charges for each byte of calldata, `Gtxdatazero` for a zero byte, and `Gtxdatanonzero` for a non-zero byte, which are 4 gas and 16 gas respectively, as shown by the yellowpaper:

![ethereum yellowpaper calldata gas cost](https://static.wixstatic.com/media/935a00_10850ce3d9db4fe1b89e64e2a5dd28b6~mv2.png/v1/fill/w_740,h_636,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_10850ce3d9db4fe1b89e64e2a5dd28b6~mv2.png)

Layer 2s post calldata to the layer 1, so they must pay the layer 1 calldata cost. Furthermore, the layer 2 imposes an added “security fee.”

Mathematically, the total layer2 transaction’s gas is defined as :

![l2 gas cost formula](https://static.wixstatic.com/media/935a00_33916a09e9264f058c27264837d87ff1~mv2.png/v1/fill/w_740,h_49,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_33916a09e9264f058c27264837d87ff1~mv2.png)

The L1 gas can often be from 90% to 99% of the total gas cost (L1 + L2 gas). It is noted that these figures heavily depend on the network `congestion` on L1.

## Different Rules for Different L2 Chains

Although it is true that most of the gas spent on L2s comes from data/security part, the same set of smart contracts on different L2s could produce different gas results. This is because different L2 chains (like Arbitrum/ Optimism/ Starknet and etc) use different rules and formulas to compute how much they will charge the user for the calldata on top of the L1 cost. So, if one gas optimization method produces the optimal on one L2 chain, it does not mean it will also produce the same optimal result on the other L2 chains.

Moreover, these rules will have been evolving, as the client & Ethereum ecosystem matures overtime. By way of illustration, EIP4844 (aka [Proto-Danksharding](https://www.eip4844.com/) ) will make gas L2 data/security component even cheaper and the L2 execution part more significant, resulting in possible changes in how L2 execution fee will be calculated in order to reflect appropriate incentive & economic model.

Here is how different L2s’ transaction gas is calculated:

### Arbitrum

The following is the formula that Arbitrum uses to calculate the gas cost of a transaction:

![arbitrum gas cost formula](https://static.wixstatic.com/media/935a00_dc9fb208e33f487db5f62f5b2a512d3a~mv2.png/v1/fill/w_740,h_555,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_dc9fb208e33f487db5f62f5b2a512d3a~mv2.png)

The `ExecutionFee` is calculated similar to how transactions are computed on an EVM chain, except that it is subject to a `PriceFloor`.

Arbitrum attempts to compress the calldata using the [Brotli algorithm](https://research.arbitrum.io/t/compression-in-nitro/20) before posting it to the L1.

### Optimism

Optimism has a slightly different model for charging for calldata:

![optimism gas cost formula](https://static.wixstatic.com/media/935a00_9ffd3b6a87b04871be7ab4c021396da4~mv2.png/v1/fill/w_740,h_555,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_9ffd3b6a87b04871be7ab4c021396da4~mv2.png)

You can think of the blue underlined terms as what Ethereum charges and the red underlined terms as Optimism’s profit margin.

## Methods of Optimizing Calldata

The key factor to determine the amount of gas required for the calldata component is the calldata size, and this is specified by the **ABI encoding** rule. In particular, the **ABI** (Application Binary Interface), according to [Solidity’s Official Documentation](https://docs.soliditylang.org/en/latest/abi-spec.html).

The best way to get an intuition for calldata formatting is with an example.

First, let's install cast, a toolkit to interact with EVM, and we use Foundryup as a toolchain installer:

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

Then we use the following cast command that shows how Solidity encodes the function with arguments:

```solidity!
cast calldata "addLiquidity(address,address,uint256,uint256,uint256,uint256,address,uint256)" 0xdeaDDeADDEaDdeaDdEAddEADDEAdDeadDEADDEaD 0xdeaDDeADDEaDdeaDdEAddEADDEAdDeadDEADDEaD 1200000000000000000000 2500000000000000000000 1000000000000000000000 2000000000000000000000 0xdeaDDeADDEaDdeaDdEAddEADDEAdDeadDEADDEaD 100
```

The result has the total bytes count of 520 hexadecimal = 520/2 = 260 bytes:

```bash!
0xe8e33700000000000000000000000000deaddeaddeaddeaddeaddeaddeaddeaddeaddead000000000000000000000000deaddeaddeaddeaddeaddeaddeaddeaddeaddead0000000000000000000000000000000000000000000000410d586a20a4c000000000000000000000000000000000000000000000000000878678326eac90000000000000000000000000000000000000000000000000003635c9adc5dea0000000000000000000000000000000000000000000000000006c6b935b8bbd400000000000000000000000000000deaddeaddeaddeaddeaddeaddeaddeaddeaddead0000000000000000000000000000000000000000000000000000000000000064
```

As you can see, the first 4 bytes of the calldata are the first four bytes of the Keccak256 hash of the function signature (addLiquidity(address,..)). After the [function selector](https://www.rareskills.io/post/function-selector), the following chunks of 32 bytes are the function arguments. If the argument is shorter than 32 bytes, it is, by default, “left padded” with extra zeroes to fit inside the 32 byte.

To illustrate, the chunks of calldata can be split as follows:

-   0xe8e33700 as **function selector**
-   000000000000000000000000deaddeaddeaddeaddeaddeaddeaddeaddeaddead as **address** of 0xdeaDDeADDEaDdeaDdEAddEADDEAdDeadDEADDEaD
-   000000000000000000000000deaddeaddeaddeaddeaddeaddeaddeaddeaddead as **address** of 0xdeaDDeADDEaDdeaDdEAddEADDEAdDeadDEADDEaD
-   0000000000000000000000000000000000000000000000410d586a20a4c00000 as **uint256** of 1200000000000000000000
-   0000000000000000000000000000000000000000000000878678326eac900000 as **uint256** of 2500000000000000000000
-   00000000000000000000000000000000000000000000003635c9adc5dea00000 as **uint256** of 1000000000000000000000
-   00000000000000000000000000000000000000000000006c6b935b8bbd400000 as **uint256** of 2000000000000000000000
-   000000000000000000000000deaddeaddeaddeaddeaddeaddeaddeaddeaddead as **address** of 0xdeaDDeADDEaDdeaDdEAddEADDEAdDeadDEADDEaD
-   0000000000000000000000000000000000000000000000000000000000000064 as **uint256** of 64

There are a bunch of techniques to reduce total bytes of calldata, without losing the information. The concept is to try to encode calldata in a compact way in order to use as little bytes of calldata as possible. Then, the encoded data is later decoded into usable format.

The overhead of decompressing the calldata is usually negligible compared to the gas saved by compressing the calldata.

The tricks discussed here will not work in every possible context. The amount of bytes saved heavily depend on specific business logic in the smart contract.

## Bypassing most often used bytes

If we know the exact value of either function signature or the function parameter, we can hard-code them as constant to be used later when required. Since we have a limited set of functions for example, we don’t need the full four bytes to identify them.

We can design the set of smart contract using factory pattern such that the factory deploys a unique contract with every combination of methods and parameters. Some gas optimization examples are provided as following:

### Function signature

We can save 4 bytes of calldata by using only fallback function in the contract:

```solidity
  fallback() external payable {
    // business logic
  }
  ```

### Function parameters

We can save 32 bytes of calldata by removing one parameter from the function

For instance, the **address** of ERC20 contract can be hardcoded as constant and could removed from the function. This can possibly save total of 20 non-zero bytes (same as **address** size ) and 12 zero bytes ( padded bytes to full 32 bytes).

```solidity
  address public constant USDC = <address>;

  function TEST() external {
    // business logic using  USDC
  }
```

If you are curious and would like to explore more about the implementation in practice you can checkout the following projects with interesting designs:

-   [Scopelift’s calldata-optimized router](https://scopelift.co/blog/calldata-optimizooooors)
-   [op-kompressor’s ERC4337](https://github.com/clabby/op-kompressor)

### Caching addresses using an address table

**AddressTable** can be thought as a cached database which stores previously registered addresses using an id.

For instance, the user registers the address first, then the address is automatically mapped to an id. Later, user can just use the id instead of the full address. This results in greatly reduced size of calldata from 20 bytes to only a few bytes.

Behind the scenes, the table is just a smart contract that store the mapping between addresses and indexes. It also has functionality look up the registered address using relevant mapped id.

This design is adopted and implemented by **Arbitrum**. The interface is as following:

```solidity!
interface ArbAddressTable {
    /**
     * @notice Check whether an address exists in the address table
     * @param addr address to check for presence in table
     * @return true if address is in table
     */
    function addressExists(address addr) external view returns (bool);

    /**
     * @notice compress an address and return the result
     * @param addr address to compress
     * @return compressed address bytes
     */
    function compress(address addr) external returns (bytes memory);

    /**
     * @notice read a compressed address from a bytes buffer
     * @param buf bytes buffer containing an address
     * @param offset offset of target address
     * @return resulting address and updated offset into the buffer (revert if buffer is too short)
     */
    function decompress(bytes calldata buf, uint256 offset)
        external
        view
        returns (address, uint256);

    /**
     * @param addr address to lookup
     * @return index of an address in the address table (revert if address isn't in the table)
     */
    function lookup(address addr) external view returns (uint256);

    /**
     * @param index index to lookup address
     * @return address at a given index in address table (revert if index is beyond end of table)
     */
    function lookupIndex(uint256 index) external view returns (address);

    /**
     * @notice Register an address in the address table
     * @param addr address to register
     * @return index of the address (existing index, or newly created index if not already registered)
     */
    function register(address addr) external returns (uint256);

    /**
     * @return size of address table (= first unused index)
     */
    function size() external view returns (uint256);
}
```

However, the implementation is a precompile contract which is written in **Go**. You can check **OffchainLabs**’s [git repository](https://github.com/OffchainLabs/nitro/blob/v2.0.14/precompiles/ArbAddressTable.go) here. It is intended to be a single universal address table where anyone can register and use it.

If you want to see another implementation written in **Solidity**, together with its application. This **Solid Grinder**’s [git repository](https://github.com/Ratimon/solid-grinder) contains modified version of [UniswapV2](https://github.com/Ratimon/solid-grinder/blob/main/contracts/examples/uniswapv2/UniswapV2Router02_Optimized.sol), which adopt its own [address table](https://github.com/Ratimon/solid-grinder/blob/main/contracts/AddressTable.sol).

### Data Serialization

**Data Serialization** works by serializing and deserializing parameters into the correct type with adequate data size.

For example, if we choose to reduce the calldata by sending the time period as arguments with type of uint40 (5 bytes) instead of uint256, the calldata should be sliced at the correct offset and the result (after zero bytes removed) can be correctly used in the next steps.

Let look at the implementation by **Solid Grinder** again [here](https://github.com/Ratimon/solid-grinder/blob/main/contracts/examples/uniswapv2/decoder/UniswapV2Router02_Decoder.g.sol). This contract is a good starting point:

![data serialization function from the solid grinder tool](https://static.wixstatic.com/media/935a00_f05871e23b4942af88b297dfa96cb22f~mv2.png/v1/fill/w_740,h_266,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_f05871e23b4942af88b297dfa96cb22f~mv2.png)

This decoder function is application-specific to **Uniswapv2** and it is generated with **Solid Grinder** 's CLI by looking at original unoptimized function. In this case, it is **UniswapV2Router02**. Basically, you can experiment and follow the detailed steps here at [Quick Start](https://github.com/Ratimon/solid-grinder/tree/main#quickstart).

## Tradeoffs

The clearest tradeoffs of above calldata gas optimization tricks are readability and complexity. For example,

Adding encode and decode logics into smart contract and explicitly removing function’s parameters will not only confuse users who directly interact with the contract via **Etherscan**, but will also make harder job for the developer who want to build on-top of your modified smart contract, reducing composability which is the unique strength of permissionless world.

## Final thoughts

Aforementioned, calldata gas optimization is a new topic, but will be becoming more recently relevant as layer 2/rollup technology are becoming more mainstream. Moreover, there is still no clear standard and practice. This article just provides and proposes possible design decisions & approaches. There is more room to reinvent this paradigm.

## References

-   [https://docs.arbitrum.io/arbos/l1-pricing#l1-fee-collection](https://docs.arbitrum.io/arbos/l1-pricing#l1-fee-collection)
-   [https://docs.arbitrum.io/stylus/reference/opcode-hostio-pricing#opcode-costs](https://docs.arbitrum.io/stylus/reference/opcode-hostio-pricing#opcode-costs)
-   [https://github.com/OffchainLabs/arbitrum-tutorials/tree/master/packages/address-table](https://github.com/OffchainLabs/arbitrum-tutorials/tree/master/packages/address-table)
-   [https://community.optimism.io/docs/developers/build/transaction-fees/](https://community.optimism.io/docs/developers/build/transaction-fees/)
-   [https://scopelift.co/blog/calldata-optimizooooors](https://scopelift.co/blog/calldata-optimizooooors)
-   [https://github.com/clabby/op-kompressor](https://github.com/clabby/op-kompressor)
-   [https://github.com/Ratimon/solid-grinder](https://github.com/Ratimon/solid-grinder)

*Originally Published January 30, 2024*
