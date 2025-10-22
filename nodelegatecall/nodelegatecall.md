# noDelegateCall Explained

The `noDelegateCall` modifier prevents [delegatecall](https://www.rareskills.io/post/delegatecall) from being sent to a contract. We will first show the mechanism for how to accomplish this then later discuss the motivation for why one might do this.

Below, we've simplified the `noDelegateCall` modifier originally created by [Uniswap V3's noDelegateCall](https://github.com/Uniswap/v3-core/blob/d8b1c635c275d2a9450bd6a78f3fa2484fef73eb/contracts/NoDelegateCall.sol#L4):

```solidity
contract NoDelegateCallExample {
    address immutable private originalAddress;

    constructor() {
        originalAddress = address(this);
    }

    modifier noDelegateCall() {
        require(address(this) == originalAddress, "no delegate call");
        _;
    }
}
```

The `address(this)` will change depending on the execution environment, but `originalAddress` will always be the deployed address of the code that uses `noDelegateCall`. So if another contract does a [delegatecall](https://www.rareskills.io/post/delegatecall) to a function with the `noDelegateCall` modifier, then `address(this)` will not equal `originalAddress` and the transaction will revert. It is extremely critical that original address is an immutable variable, otherwise the contract issuing the delegatecall could strategically put the address of the contract using noDelegateCall in that slot and bypass the `require` statement.

## Testing noDelegateCall

Below we provide the code to test `noDelegateCall`.

```solidity
contract noDelegateCall {
    address immutable private originalAddress;

    constructor() {
        originalAddress = address(this);
    }

    modifier noDelegateCall() {
        require(address(this) == originalAddress, "no delegate call");
        _;
    }
}

contract A is noDelegateCall {
    uint256 public x;

    function increment() noDelegateCall public {
        x++;
    }
}

contract B {
    uint256 public x; // this variable does not increment

    function tryDelegatecall(address a) external {
        (bool ok, ) = a.delegatecall(
            abi.encodeWithSignature("increment()")
        );// ignore ok
    }
}
```

Contract `B` does a delegatecall to `A` which is using the `noDelegateCall` modifier. Although the transaction to `B.tryDelegatecall` will not revert because the return value of the [low level call](https://www.rareskills.io/post/low-level-call-solidity) is ignored, the storage variable `x` will not be incremented because the transaction inside the context of delegatecall reverts.

## Motivation for noDelegateCall

[Uniswap V2](https://rareskills.io/uniswap-v2-book) is the most forked DeFi protocol in history. The Uniswap V2 protocol faced competition from projects copying the source code line-by-line and marketing the new product as an alternative to Uniswap V2 and sometimes incentivizing users by providing airdrops.

To prevent this from happening, the Uniswap team licensed Uniswap V3 under the [Business Source License](https://support.uniswap.org/hc/en-us/articles/14569783029645-Uniswap-v3-Licensing) — anyone can copy the code but it cannot be used for commercial purposes until the license expired in April 2023.

However, if someone wanted to make a "copy" of Uniswap V3 they could simply create a [clone proxy](https://www.rareskills.io/post/eip-1167-minimal-proxy-standard-with-initialization-clone-pattern) and point it to an instance of Uniswap V3 — then market that smart contract as an alternative to Uniswap V3. The `noDelegateCall` modifier prevents this from happening.

*Originally Published May 11*
