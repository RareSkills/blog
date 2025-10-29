# Solidity RSA signatures for airdrops and presales: Beating ECDSA and Merkle Trees in Gas Efficiency

Updated: Aug 4, 2023

By [Suthan Somadeva](https://twitter.com/peakycryptos) and [Michael Burke](https://twitter.com/@ThisIsMikeDB)

## ECDSA Vs RSA Introduction

Creating a contract that gives certain addresses permissions goes by several names: airdrops, presale, whitelist, allowlist, and so forth. But the essence of the idea is there is a set of addresses that have a special permission to buy a high demand token for a desirable price (sometimes free).

There are three established solutions for this: mappings, Merkle trees, and ECDSA signatures.

The relative merits of these approaches have already been [discussed elsewhere](https://medium.com/donkeverse/hardcore-gas-savings-in-nft-minting-part-2-signatures-vs-merkle-trees-917c43c59b07), so we will summarize the results:

- ECDSA Signature Verification (Gas: 29,293)
- using Merkle proofs (Gas: 30, 517, 128 addresses)

A mapping presale works by the seller entering the addresses of the customers into a mapping which goes from address to boolean to note if the address is privileged or not. The mapping is set to false after the buyer completes the transaction. This ensures only addresses that are marked true can make a purchase. This is very gas efficient for the buyer, but the seller could [spend millions of gas](https://jeffrey-scholz.medium.com/hardcore-gas-savings-in-nft-minting-part-3-save-30-000-in-presale-gas-c945406e89f0) allowlisting thousands of addresses, (it will cost at least 22,100 gas to add an address to the allowlist).

Because of this, Merkle Trees and ECDSA signatures (using the [ethereum precompile](https://www.rareskills.io/post/solidity-precompiles) ecerecover, or better yet, the [more secure wrapper around it](https://docs.openzeppelin.com/contracts/2.x/api/cryptography#ECDSA) published by OpenZeppelin), are often preferred to mappings. The gas cost (for the buyer) of executing the ECDSA signature verification is 29,293 gas. This includes the 21,000 to initiate the transaction, so the ECDSA cost is 8,293. Note that this includes reading the signing address from storage, but this cost is necessary or we won’t be able to invalidate signatures.

Merkle Trees vary in cost based on tree size (larger trees require larger Merkle proofs), but if over 1,000 addresses are in the Merkle tree, it will cost at least 32,000 gas (or more) to verify the address. This cost is clearly inferior to ECDSA.

The goal then, is to beat ECDSA which costs 8,293 gas. To keep solutions apples-to-apples, the alternative solution must:

1. Be able to nullify allowlisted addresses. Merkle trees can change their root, ECDSA , can change their signing address, and mappings can set the value to false.
2. Not impose a substantial cost burden on the seller the way mappings do
3. Cost less than 8,200 gas for the buyer (including a storage load)
4. Not be compromised on security

## Prerequisites

To understand the proposed methodology, the reader should be familiar with the following subjects:

- Precompiled smart contracts ([recommended reading](https://www.evm.codes/precompiled?fork=grayGlacier), [stackexchange](https://ethereum.stackexchange.com/questions/15479/list-of-pre-compiled-contracts))
- Metamorphic contracts ([introduction](https://blog.openzeppelin.com/the-state-of-smart-contract-upgrades/#metamorphic-contracts), [recommended reading](https://a16zcrypto.com/metamorphic-smart-contract-detector-tool/))
- Access lists ([recommended reading](https://hackmd.io/@fvictorio/gas-costs-after-berlin#EIP-2930-Optional-Access-List-transactions))
- How to use digital signatures and their appropriate use cases in blockchain applications
- How gas cost is computed, especially as it relates to calldata, opcodes, storage, and memory usage.

## The RSA Algorithm

If we want to substantially beat ECDSA, we need to find a different cryptographic algorithm that allows for the proof of set membership. ECDSA is actually the newer, cooler version of the original digital signature algorithm, RSA. ECDSA relies on discrete logarithms over elliptic curves being hard (hence the name – elliptic curve digital signature algorithm). RSA (named after its authors, Rivest, Shamir, Adleman) relies on large integers being hard to factor. In terms of age, RSA was published in the 1970s, but ECDSA became a formal specification in the early 2000s.

![Admiral Piett approves of RSA cryptography](https://static.wixstatic.com/media/b09095_e8b9d06bc1134d3f84d29dd2e83c4ce5~mv2.png/v1/fill/w_728,h_493,al_c,lg_1,q_85,enc_auto/b09095_e8b9d06bc1134d3f84d29dd2e83c4ce5~mv2.png)
Admiral Piett approves of RSA cryptography

We will not explain RSA in detail here, but some prerequisites are in order.

The signer picks two large prime numbers `p` and `q`, and multiplies them together to produce `n`. This `n` is the first part of the public key. Second, the signer picks a small prime `e` (can be hardcoded to 3 for our use case), and publishes the pair (`n`, `e`) as the public key. Behind the scenes, the signer computes

```solidity
t = (p - 1) * (q - 1)
d = t^(-1) % n
```

The number `d` is the private key. If an adversary could decompose `n` into `p` and `q`, then computing `d` would be trivial. But it is known that [integer factorization is hard](https://en.wikipedia.org/wiki/Integer_factorization). To sign a message, the signer hashes the message `m` to get `h` and raises `h` to the power `d`. That is,

```solidity
s = h(m) ^ d % n
```

The signer then publishes (`m`, `s`) as the message and signature.

The verifier then hashes `m` and raises it to the power `e mod n`. Remember, `e` and `n` are the public key. If and only if

```solidity
s == s ^ e % n
```

then the signature is valid for the public key (`n`, `e`). Note that if `n` is extremely large, then the probability that `s == s ^ e % n` by random chance is vanishingly small. If the equality checks out, then we know the signature is valid for the public key.

To do this in Ethereum, we would simply sign an address as

```solidity
s = buyerAddress ^ d % n
```

and the smart contract would verify

```solidity
msg.sender == s ^ e % n
```

## Using only 256 bits is unsafe

We say “factoring integers is hard” but clearly this implies the number must be large enough. For example, 33 is composed of two prime numbers and is trivial to decompose. Thankfully, we have real world benchmarks of what the state-of-the-art can accomplish in the [RSA Factoring Challenge](https://en.wikipedia.org/wiki/RSA_Factoring_Challenge).

To clarify, the number of “bits” refers to the size of the public key. Therefore, when we say “RSA 2048”, this means the number n, the modulus, has 2048 bits. When comparing key sizes, it’s important to remember that each additional bit doubles the size of the number. So 700 bits is exponentially more secure than 350 bits, not twice as secure.

The largest key to be cracked (factored) so far had 829 bits, and it required a modern supercomputer to do it. The team utilized approximately 2700 CPU core-years, using a 2.1 GHz Intel Xeon Gold 6130 CPU. The [cheapest 16 core CPU on AWS](https://aws.amazon.com/ec2/pricing/on-demand/) costs \$0.40 cents per hour, so the cost to crack this key is on the order of \$9.4 million dollars. Even assuming generous discounts from the cloud provider, the cost is in the millions.

## Modular arithmetic with more than 256 bits in Solidity

Ethereum only supports 32 byte data types, so by default, we cannot carry out

```solidity
s ^ e % n
```

Thankfully, the ethereum [blockchain](https://www.rareskills.io/web3-blockchain-bootcamps) added a precompiled contract in [EIP 198](https://eips.ethereum.org/EIPS/eip-198) specifically to support modular arithmetic. To use it, the base, exponent, and modulus must be loaded into memory in [abi encoded](https://www.rareskills.io/post/abi-encoding) format. Then the contract living at address 0x05 is invoked.

Storing the public key becomes a bit of an issue if you use a secure amount of bits. If the key size is 1024 bits, that necessitates 4 storage slots. To read the public key out of storage would be four SLOAD operations, for a total of 8,400 gas. This by itself is already less efficient than the ECDSA solution benchmarked above.

If we use immutable variables, this cost is largely eliminated, but this creates a weakness if we cannot retroactively remove someone from the presale. In traditional ECDSA or Merkle Trees, we would just replace the signing address or the Merkle Root. This isn’t possible if we use an immutable variable.

However, storing the public key in the bytecode instead of storage is the key idea. Reading an external contract’s bytecode ([EXTCODECOPY](https://ethervm.io/#3C)) costs 2,600 gas, which is far less than the 8,400 gas of reading each part of the public key in four pieces.

To invalidate a public key, we could simply create a new contract, and update a storage variable to point to the new address. But that adds back an additional 2,100 gas.

It turns out, it is possible to store the external contract’s address (whose bytecode stores the public key) in an immutable variable, but still invalidate the public key by mutating the external smart contract’s bytecode.

## Invalidating the public key with the metamorphic contract pattern

A smart contract is created by loading the [deployment bytecode](https://monokh.com/posts/ethereum-contract-creation-bytecode) into memory, then returning the range of the bytecode which contains the runtime code.

When a smart contract is created with the create2 command, the address of the contract can be predicted in advance. The address is calculated from the combination of a salt, the deployer, and the initialization bytecode. If the contract selfdestructs, then a new contract at the same address can be deployed.

Note that the address is a function of the initialization code, not the deployed code. It is possible to deploy different bytecode by having the initialization code load a different runtime code. By selfdestructing and redeploying with the same initialization code but different deployment code, the bytecode of the contract can mutate.

Therefore, we can have a smart contract whose first *k* bytes are for selfdestructing under a certain condition, but the rest of the bytes are just the RSA public key.

Because the address is determined in advance, we can store the address of this metamorphic contract in an immutable variable. We can EXTCODECOPY from that address when we need the public key. To replace the public key, we instruct the contract to selfdestruct, then deploy a new contract to that address.

## Saving an additional 100 gas with access lists

[EIP 2930](https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum) added a new type of transaction that allows the user to specify in advance which addresses and storage slots will be accessed. This allows the nodes to prefetch those values from storage, thus speeding up execution time. Using an access list transaction when calling an external contract can save 100 gas. Note that this saving does not apply when a smart contract accesses its own storage variable. Because this RSA presale airdrop design relies on an external contract to store the public key, then using an access list is appropriate.

## Benchmarks: Gas cost vs key size

Most of the gas cost comes from having very large calldata as a result of large signatures. If the key size is set to 1024 bits, then the calldata will be 128 bytes. Each byte costs 16 gas, so the total gas cost just to have calldata that large is 2,048 gas.

Compared to most other use cases, ours uses a considerable amount of memory and Ethereum charges for this.

## Why this saves gas relative to ECDSA

The benchmarks make clear that the larger the key (and hence the signature), the larger the gas cost. Our design exploits the fact that the precompile assigns a low price to raising numbers to a low power. The cost of executing the precompiled contract under those conditions at 0x05 is only a few hundred gas compared to thousands for executing the precompile for ECDSA.

## Choosing a key size

Although a key with 829 bits has been factored, it requires a modern supercomputer to do so. For applications such as airdropping lower value tokens or putting NFTs on a presale, an attacker does not have an incentive to spend six figures to a million dollars to factor a public key and obtain an NFT in a presale. The most expensive Ethereum tokens at the time of writing ([Fidenza](https://opensea.io/collection/fidenza-by-tyler-hobbs) and [Bored Ape Yacht Club](https://opensea.io/collection/boredapeyachtclub)) cost approximately \$100,000 a piece, so for the vast majority of applications, it is not economical for an attacker to try to factor the public key.

Remember, each bit doubles the difficulty of factoring the integer, so as of 2022, most low value token presales are probably safe with 896 bits. In this case, saving users over 2,500 gas compared to ECDSA is compelling.

## Key Size Benchmarks

- RSA-896 (Gas: 26,850)
- RSA-960 (Gas: 26,925)
- RSA-1024 (Gas: 27,033)
- RSA-2048 (Gas: 29,271)

## Conclusion

We present a novel method for assigning addresses to a presale or airdrop that is more efficient than the known solutions today. By combining the modular exponentiation precompile with the metamorphic contract pattern and the access list transaction, we can validate secure RSA signatures on-chain in a gas efficient manner.

Our work is only a proof of concept at this stage. Developers are advised to use caution using the code in production.

This project was created by [Suthan Somadeva](https://twitter.com/peakycryptos) and [Michael Burke](https://twitter.com/ThisIsMikeDB) as part of the [RareSkills Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp). The code implementation and unit tests are on [github](https://github.com/RareSkills/RSA-presale-allowlist).

*Originally Published Dec 7, 2022*
