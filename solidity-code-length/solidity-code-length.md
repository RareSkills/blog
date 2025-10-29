# Three ways to detect if an address is a smart contract

This article describes three methods in Solidity for determining if an address is a smart contract:

-   Check if `msg.sender == tx.origin`. This is not a recommended method, but because many smart contracts use it, we discuss this method for completeness.
-   The second (and recommended way) is to measure the bytecode size of the address using `code.length`. This approach still has limitations that devs must work around.
-   The third is using `codehash` and is not recommended because it has the same limitations as `code.length` with additional complexity.

We discuss each method in this tutorial. Finally, we provide some Solidity puzzles at the end to test your understanding.

## Method 1: Using msg.sender == tx.origin to detect if an address is a smart contract

The global variable `tx.origin` is the wallet that initiated the transaction, while `msg.sender` is the address that called the smart contract. If a wallet calls a smart contract directly, then `tx.origin` will be the same as `msg.sender`.

However, suppose a wallet calls smart `contract A` which then calls smart `contract B`.

From `contract B`’s perspective, `msg.sender` is `contract A` and the wallet is `tx.origin`. Clearly, `msg.sender` will not equal `tx.origin` inside of `contract B`. The diagram below illustrates the relationship:

![A Diagram showing the msg.sender and tx.origin when a wallet directly calls a contract and when a wallet calls a smart through another smart contract.](https://static.wixstatic.com/media/706568_0217596ae80e4414934e26a71ebe5dad~mv2.jpg/v1/fill/w_592,h_444,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/706568_0217596ae80e4414934e26a71ebe5dad~mv2.jpg)

By checking if `msg.sender == tx.origin`, the smart contract can detect if the incoming call is from a smart contract or from a wallet.

### require(msg.sender == tx.origin) is an antipattern

Using a smart contract as a wallet is becoming increasingly popular with the adoption of account abstraction, such as ERC-4337 and using smart contracts for multisignature wallets (like Gnosis Safe).

Adding `require(msg.sender == tx.origin)` to a smart contract means that account abstraction wallets and multisignature wallets cannot interact with the smart contract.

This technique can only test if `msg.sender` is a contract or not. It cannot test an arbitrary address.

## Method 2: Detecting if an address is a smart contract with **`code.length`**

The recommended way for a smart contract to test if an address is a smart contract is to measure the size of its bytecode.

**If an address has bytecode, then it is a smart contract.**

Consider the following code:

```solidity!
contract TestAddress {
    function test(
        address target
    )
    public
    view
    returns (bool isContract) {
        if (target.code.length == 0) {
            isContract = false;
        } else {
            isContract = true;
        }
    }
}
```

Although all smart contracts have bytecode and all wallet addresses do not, there are some “gotchas” to keep in mind:

-   An address which has no bytecode now could have bytecode drop there in the future if a smart contract gets deployed to that address.
-   Using `msg.sender.code.length == 0` is not a reliable way to detect if an _incoming_ call is from a smart contract. If a smart contract makes a call <u>from the constructor</u> then it has not deployed its bytecode yet and `msg.sender.code.length` will be 0. While the constructor is executing, the bytecode of the smart contract has not yet been deployed. Therefore, `code.length` will be zero.
-   On EVM chains that support `selfdestruct`, there might have been a smart contract at `target` in the past, but the smart contract self destructed.

### Testing msg.sender with code.length

<u>If a _wallet_ calls a contract</u>, then `msg.sender.code.length` is guaranteed to be 0.

<u>If a _contract_ calls a another contract</u>, then `msg.sender.code.length` will be 0 if called from the constructor and non-zero if called from another smart contract function.

### Testing an address (not msg.sender) with code.length

If a smart contract uses the `address(target).code.length` test on some `target`, <u>and the target is a smart contract</u>, then `address(target).code.length` is guaranteed to be non-zero.

The dev should keep in mind the `code.length` could become 0 later if the contract self destructs (assuming the chain supports selfdestruct and the contract has the ability to selfdestruct).

If a smart contract uses the `address(target).code.length` test on some `target`, <u>and the target is a wallet</u>, then `address(target.code.length)` is guaranteed to be 0.

However, just because `address(target).code.length` is 0 now does not mean it will always be zero. A smart contract might be deployed there later. Suppose I give you an address. You measure it now with `address(target).code.length` and it returns 0. That measurement will be accurate at the moment you measured it, but it is possible that I could deploy a contract to that address (`target`) at a later date and if you measure it again with `address(target).code.length` it will be non-zero.

### Common use case for checking if an address is a smart contract

If a token is transferred to a smart contract that does not have the functionality to send the tokens out, then the tokens will remain stuck, owned by that contract forever.

As such, some token standards take steps to prevent this from happening.

The [ERC-721](https://www.rareskills.io/post/erc721) with the `safeTransferFrom` function for example will check if the address being transferred to is a smart contract (using the `code.length` trick).

![A code snippet of the checkOnERC721Received function highlighting the code.length if statement](https://static.wixstatic.com/media/706568_55ba9c2653f8419a83912b74f96dc4fc~mv2.png/v1/fill/w_592,h_372,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_55ba9c2653f8419a83912b74f96dc4fc~mv2.png)

([original code](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/utils/ERC721Utils.sol#L22))

If it is, they attempt to call a special function on the contract to ask if the contract supports ERC-721 tokens. If the function isn’t there, then it knows the tokens will get stuck and it blocks the transfer.

## Method 3: Codehash is a bad way to test if an address is a contract

The `codehash` returns the `keccak256` of the bytecode of an address.

It has the following behavior:

-   If the address has no Ethereum balance and no bytecode, there is nothing to hash and returns `bytes32(0)`.
-   If the address has an Ethereum balance but no bytecode, it returns the keccak256 of empty data `keccak256("")` which equals `0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`.
-   If the address has bytecode (regardless of balance) it returns the `keccak256` of the bytecode of the contract.

The exact behavior of codehash is described in the [Etherum client comments on codehash](https://github.com/ethereum/go-ethereum/blob/7bb3fb1481acbffd91afe19f802c29b1ae6ea60c/core/vm/instructions.go#L391-L416).

Some contracts have mistakenly used the `codehash` to test if an address has bytecode or not. This is not a good idea because if we use `codehash` on a contract that has no bytecode, we will either get back `bytes32(0)` or `keccak256("")` and we must check both possibilities.

If an address `a` has no bytecode **and has no ether**, then `address(a).codehash` returns `bytes32(0)` or 32 bytes of all zero. However, if someone transfers Ether to the address, then the `codehash` will become `keccak256("")`, despite not being a wallet and not a smart contract.

You can [test the code below in Remix](https://remix.ethereum.org/#code=Y29udHJhY3QgVGVzdEhhc2ggewoKICAgIGZ1bmN0aW9uIGdldEhhc2goKQogICAgICAgIGV4dGVybmFsCiAgICAgICAgdmlldwogICAgICAgIHJldHVybnMgKGJ5dGVzMzIpIHsKICAgICAgICAgICAgLy8gcmFuZG9tIGFkZHJlc3Mgd2l0aCBubyBiYWxhbmNlIG9yIGNvZGUKICAgICAgICAgICAgcmV0dXJuIGFkZHJlc3MoMTAxKS5jb2RlaGFzaDsKICAgICAgICAgICAgLy8gcmV0dXJucyAweDAwMC4uLjAwMAogICAgfQoKICAgIGZ1bmN0aW9uIGhhc2hPZk5vbkVtcHR5V2FsbGV0KCkKICAgICAgICBleHRlcm5hbAogICAgICAgIHZpZXcKICAgICAgICByZXR1cm5zIChieXRlczMyKSB7CiAgICAgICAgICAgIC8vIHR4Lm9yaWdpbiBoYXMgYSBub24temVybyBldGhlciBiYWxhbmNlCiAgICAgICAgICAgIHJldHVybiB0eC5vcmlnaW4uY29kZWhhc2g7CiAgICAgICAgICAgIC8vIHJldHVybnMgYSBub24temVybyBoYXNoCiAgICB9CgogICAgLy8gb2JzZXJ2ZSB0aGF0IGBrZWNjYWtOaWxgIGFuZCBgaGFzaE9mTm9uRW1wdHlXYWxsZXRgCiAgICAvLyByZXR1cm4gdGhlIHNhbWUgdmFsdWUKICAgIGZ1bmN0aW9uIGtlY2Nha05pbCgpIGV4dGVybmFsIHB1cmUgcmV0dXJucyAoYnl0ZXMzMikgewogICAgICAgIHJldHVybiBrZWNjYWsyNTYoIiIpOwogICAgfQoKICAgIC8vIERlcGxveSBTb21lVGVzdENvbnRyYWN0IGFuZCBwdXQgaXRzIGFkZHJlc3MgaW4KICAgIC8vIGNvZGVIYXNoT3RoZXJDb250cmFjdCB0byB0ZXN0IGl0CiAgICBmdW5jdGlvbiBjb2RlSGFzaE90aGVyQ29udHJhY3QoYWRkcmVzcyBfYSkgZXh0ZXJuYWwgdmlldyByZXR1cm5zIChib29sKSB7CiAgICAgICAgLy8gcmV0dXJucyB0cnVlIGJlY2F1c2UgdGhlIGNvZGVoYXNoIG9mIGFub3RoZXIgY29udHJhY3QKICAgICAgICAvLyBpcyBlcXVhbCB0byB0aGUgYGtlY2NhazI1NmAgb2YgaXRzIGJ5dGVjb2RlCiAgICAgICAgcmV0dXJuIF9hLmNvZGVoYXNoID09IGtlY2NhazI1NihfYS5jb2RlKTsKICAgIH0KfQoKY29udHJhY3QgU29tZVRlc3RDb250cmFjdCB7CiAgICBmdW5jdGlvbiBzb21lRnVuY3Rpb24oKSBleHRlcm5hbCBwdXJlIHJldHVybnMgKHVpbnQyNTYpIHsKICAgICAgICByZXR1cm4gNTsKICAgIH0KfQ&lang=en&optimize=false&runs=200&evmVersion=null&version=soljson-v0.8.25+commit.b61c2a91.js) to see the behavior of codehash:

```solidity!
contract TestHash {
    function getHash()
         external
         view
         returns (bytes32) {
            // random address with no balance or code
            return address(101).codehash;// returns 0x000...000
    }

    function hashOfNonEmptyWallet()
        external
        view
        returns (bytes32) {
            // tx.origin has a non-zero ether balance
            return tx.origin.codehash;
             // returns a non-zero hash
    }

    // observe that `keccakNil` and `hashOfNonEmptyWallet`
    // return the same value
    function keccakNil()
        external
        pure
        returns (bytes32) {
            return keccak256("");
    }

    // Deploy SomeTestContract and put its address in
    // codeHashOtherContract to test it
    function codeHashOtherContract(
        address _a
    )
        external
        view
        returns (bool) {
            // returns true because the codehash
            // of another contract
            // is equal to the `keccak256` of its bytecode
            return a.codehash == keccak256(a.code);
    }
}

contract SomeTestContract {
    function someFunction()
        external
        pure
        returns (uint256) {
            return 5;
    }
}
```

Both `codehash` and `code.length` can be used to determine if an address is a smart contract by checking for the presence of bytecode; however, `codehash` introduces unnecessary complexity by hashing the bytecode, resulting in three possible outcomes, whereas we only need to check `code.length` is zero or not.

It is far simpler to check `code.length`.

## Puzzle to test your knowledge

### Puzzle 1

Can you get the following contract to return true when `puzzle` is called and not revert?

```solidity!
contract Puzzle {
    function puzzle()
        external
        view
        returns (bool success) {
            require(msg.sender != tx.origin);
            require(msg.sender.code.length == 0);
            success = true;
    }
}
```

### Puzzle 2

What should `tx.origin.code.length` return? Does it always return the same value?

## Learn more with RareSkills

See our [Solidity course](https://www.rareskills.io/learn-solidity) if you are new to Solidity. See our [Solidity bootcamp](https://www.rareskills.io/solidity-bootcamp) if you already have some experience. Thanks for reading!

*Originally Published April 5, 2024*
