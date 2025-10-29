# Ethereum precompiled contracts

Ethereum precompiles behave like smart contracts built into the Ethereum protocol. The nine precompiles live in addresses 0x01 to 0x09.

The utility of precompiles falls into four categories
- Elliptic curve digital signature recovery
- Hash methods to interact with bitcoin and zcash
- Memory copying
- Methods to enable elliptic curve math for zero knowledge proofs

These operations were deemed desirable enough to have gas-efficient mechanisms for doing them. Implementing these algorithms in Solidity would be considerably less gas efficient.

Precompiles do not execute inside a smart contract, they are part of the Ethereum client specification. You can see a list of them here in the [Geth Client](https://github.com/ethereum/go-ethereum/blob/master/core/vm/contracts.go#L81). Because they are a protocol specification, they are listed in the [Ethereum Yellow Paper](https://ethereum.github.io/yellowpaper/paper.pdf) (in Appendix E).

## Call Precompiled Smart Contracts with Solidity

Most precompiles don't have a Solidity wrapper (with ecRecover being the sole exception). You'll need to call the address directly with addressOfPrecompile.staticcall(...) or use assembly.


Although none of the precompiled contracts are state changing, the Solidity function that calls them cannot be pure because the Solidity compiler has no way of inferring that a [staticcall](https://www.rareskills.io/post/solidity-staticcall) won’t change the state.

## Address 0x01: ecRecover

ECRecover is the precompile for recovering an address from a hash and a digital signature for that hash, i.e. determining who signed it if the signature is valid. (Learn more about how to use [Solidity digital signatures](https://www.rareskills.io/post/openzeppelin-verify-signature) in our tutorial).

Example:
```solidity
function recoverSignature(bytes32 hash, uint8 v, bytes32 r, bytes32 s) public view returns (address) {
    address r = ecrecover(hash, v, r, s);
    require(r != address(0), "signature is invalid");
}
```
**Beware**: ecrecover does not revert when the signature does not validate against the hash. It returns the zero address. You should always explicitly check this, or better yet, use the Openzeppelin library that handles this for you. There are several things that can go wrong with signatures if you don’t know what you are doing!

## Address 0x02 and 0x03: SHA-256 and RIPEMD-160

Both of these precompiles will hash the bytes supplied in the calldata. Here is an example with SHA256. For the sake of simplicity, we’ll hash a uint256

```solidity
function hashSha256(uint256 numberToHash) public view returns (bytes32 h) {
    (bool ok, bytes memory out) = address(2).staticcall(abi.encode(numberToHash));
    require(ok);
    h = abi.decode(out, (bytes32));
}
```

And here it is with RIPEMD-160

```solidity
function hashRIPEMD160(bytes calldata data) public view returns (bytes20 h) {
    (bool ok, bytes memory out) = address(3).staticcall(data);
    require(ok);
    h = bytes20(abi.decode(out, (bytes32)) << 96);
}
```

Although RIPEMD-160 returns 20 bytes, the EVM can only work in 32 byte increments, which is why the bitshifting and casting is used in the example code above.

Why does Ethereum support SHA-256 and RIPEMD-160? Bitcoin makes heavy use of SHA256 the way Ethereum makes heavy use of keccak256. However, Bitcoin addresses use RIPEMD-160 to hash the public key and make the public address more compact. This is comparable to how Ethereum takes the last 20 bytes (160 bits, like RIPEMD) of the keccak256 of the [ECDSA](https://www.rareskills.io/post/solidity-rsa-signatures-for-aidrops-and-presales-beating-ecdsa-and-merkle-trees-in-gas-efficiency) public key.

## Using Yul Assembly

Because the return size is known in advance, there is no need to use the returndatasize opcode. In Yul (and in the opcode) staticcall takes six arguments:
- args
- gas to forward
- where in memory to look for the data to hash
- size of data to hash(32 bytes)
- where to write output
- size of the output

In the code below, we write the [uint256](https://www.rareskills.io/post/uint-max-value-solidity) to memory then pass it to address 2 for hashing.

```solidity
function hashSha256Yul(uint256 numberToHash) public view returns (bytes32) {
    assembly {
        mstore(0, numberToHash) // store number in the zeroth memory word

        let ok := staticcall(gas(), 2, 0, 32, 0, 32)
        if iszero(ok) {
            revert(0,0)
        }
        return(0, 32)
    }
}

```

## Address 0x04: Identity

The identity precompile copies one region of memory to another. Ethereum doesn’t have a ''memcopy'' opcode (an opcode to copy one region in memory to another). Normally, you’d have to MLOAD a word of memory onto the stack and then MSTORE it to copy it, and you’d have to do the copy word by word. With the identity precompile, you can copy a contiguous set of 32 byte words in one go, rather than one byte at a time.

## Address 0x05: Modexp

ECDSA doesn’t support public encryption. If an application has a use case for this, then good old-fashion RSA encryption must be used. At a high level, RSA works by taking a message, raising it to the power of the recipient’s public key modulo some very large number. The resulting number is the encrypted message. Since this severely limits the message’s length, the typical message exchange works by encrypting a symmetric key such as AES-256 and sending that to the recipient. Then the recipient can use the AES-256 key to decrypt the message.

Signing messages with RSA works in reverse. The sender raises the hash of the message to the power of their private key modulo the large number (which is publicly known). The result is the signature of the message. The receiver can verify the signature by raising the signature to the power of the public key modulo the large number and seeing it results in the message hash.

Ethereum does not have a public key infrastructure for RSA. However, an Ethereum address could prove ownership of an RSA public key by RSA signing their Ethereum address. Note this doesn’t work in reverse. ECDSA signing an RSA public key isn’t secure because anyone can ECDSA sign an arbitrary string, including RSA public keys.

You can see an application for [RSA with Solidity](https://www.rareskills.io/post/solidity-rsa-signatures-for-aidrops-and-presales-beating-ecdsa-and-merkle-trees-in-gas-efficiency) on our other article on the subject.

Here is an example of using `modExp` with uint256 in Solidity:
```solidity
function modExp(uint256 base, uint256 exp, uint256 mod) public view returns (uint256) {
    bytes memory precompileData = abi.encode(32, 32, 32, base, exp, mod);
    (bool ok, bytes memory data) = address(5).staticcall(precompileData);
    require(ok, "expMod failed");
    return abi.decode(data, (uint256));
}
```

## Address 0x06 and 0x07 and 0x08: ecAdd, ecMul, and ecPairing (EIP-196 and EIP-197)

These precompiles are used to make [zero knowledge proof cryptography](https://www.rareskills.io/zk-book) more efficient. In fact, you can see all three precompiles being used in the [Tornado Cash](https://www.rareskills.io/post/how-does-tornado-cash-work) zero knowledge proof verifier:


Elliptic Curve Addition: [staticcall to address(6)](https://github.com/tornadocash/tornado-core/blob/master/contracts/Verifier.sol#L79)
Elliptic Curve Multiplication: [staticcall to address(7)](https://github.com/tornadocash/tornado-core/blob/master/contracts/Verifier.sol#L100)
Elliptic Curve Pairing: [static call to address(8)](https://github.com/tornadocash/tornado-core/blob/master/contracts/Verifier.sol#L143)


These operations only support the BN-128 Barreto-Naehrig elliptic curves. These are not the same as the Elliptic curves used for digital signatures.

Ecadd and ecMul were added in EIP-196 [EIP-196](https://eips.ethereum.org/EIPS/eip-196) and ecPairing was added in [EIP-197](https://eips.ethereum.org/EIPS/eip-197).

You can learn how these precompiles work in our other tutorials:

[Elliptic Curves in Finite Fields](https://www.rareskills.io/post/elliptic-curves-finite-fields)
[Bilinear Pairings](https://www.rareskills.io/post/bilinear-pairing)

### Gas Costs for ecAdd, ecMul, and ecPairing

The gas costs for these precompiles were lowered from their original specifications with the introduction of [EIP-1108](https://eips.ethereum.org/EIPS/eip-1108). Users should refer to that EIP for up-to-date information about their gas costs instead of the respective EIP specifications.

## Address 0x09: Blake2 (EIP-152)

The Blake2 hash is the preferred hash of [zcash](https://z.cash/). Similar to SHA256 and RIPEMD-160, Blake2 was added to enable Ethereum to validate claims about transactions on that blockchain. This precompile was added in [EIP-152](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-152.md) and some sample code is available on the proposal.

## Address 0xa: Point evaluation precompile (EIP-4844)

The Decun hardfork added a precompile at address 10 (address 0xa) [precompile at address 10 (address 0xa)](https://eips.ethereum.org/EIPS/eip-4844#point-evaluation-precompile) for verifying KZG commitments. That is, given a blob commitment and a zero knowledge proof, the precompile reverts if the proof is invalid.

## Precompiles on other chains

Smart contract developers should be careful when copying Solidity code to other EVM compatible chains as the precompiles on those chains might not match what Ethereum has. For example, ecrecover and the other cryptographic precompiles are not supported on zksync. (The technical reasons for this is that most cryptography algorithms are not SNARK-friendly, they are expensive to verify from a zero knowledge proof perspective).

## Learn More

This material is part of our [Solidity bootcamp](https://www.rareskills.io/solidity-bootcamp). You can also learn Solidity for free with our free [Solidity course](https://www.rareskills.io/learn-solidity).

*Originally Published April 16, 2023*
