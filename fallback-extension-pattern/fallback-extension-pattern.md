# The Fallback Extension Pattern

The fallback-extension pattern is a simple way to circumvent the 24 KB smart contract size limit.

Suppose we have functions `foo()` and `bar()` in our **primary** contract and wish to add `baz()` but cannot due to a lack of space.

We add a fallback function to our **primary** smart contract that delegates unknown function calls to an _extension_ contract similar to how a proxy works.

We put `baz()` in the _extension_ contract. When we call `baz()` on the main contract, it will not match any of the [function selectors](https://www.rareskills.io/post/function-selector) in the **primary** contract, and thus trigger the fallback function. Then, `baz()` will be [delegatecalled](https://www.rareskills.io/post/delegatecall) in the **_extension_** contract.

## Ensuring an identical storage layout

For this pattern to work, both the primary contract and the extension contract need an identical storage layout. A simple way to accomplish this is to put _all_ the storage variables (no exceptions!) into a single contract. Then both the primary contract and the extension contract inherit from that. Here is an example:

![An image of a pattern](https://static.wixstatic.com/media/935a00_a7da77bd1f83444d97d233e0b8de36a5~mv2.png/v1/fill/w_740,h_795,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_a7da77bd1f83444d97d233e0b8de36a5~mv2.png)

## Changing the extension

In the example above, the address of the extension contract is stored in an immutable variable. We could add an additional storage variable to the storage contract which holds the address to the extension, then update the extension address when we want to change some of the functionality of the contract.

This approach is not recommended — if upgradeability is needed, then it’s better to use an established [proxy pattern](https://www.rareskills.io/proxy-patterns). Additionally, reading that extension address variable from storage will cost an extra 2,100 gas.

## Care must be taken to avoid function selector collisions

There is an approximately 1 in 4 million chance two random functions will have the same selector. However, due to the birthday paradox, this probability increases rapidly when we have _n_ function selectors and only one collision is needed to cause unwanted behavior. There are no tools for this, the developer must manually check the function selectors.

## Gas Considerations

The extension itself can follow this pattern and send it to another delegate. In fact, there is no in principal limit to how many times we do this.

However, each “hop” adds an extra 2,600 gas (the minimum gas required to issue a `CALL` or `DELEGATECALL` to a new address), so the cost can be substantial if the chain is long.

Because functions in the extension cost an extra 2,600 gas, we want to put rarely-used functions in the extension, or functions that are intended to be mostly called from off-chain where gas doesn’t matter.

### Use EIP 2930 in combination with this pattern

Using [access list transactions](https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum) with this pattern will save 100 gas when calling functions in the extension. In general, if an Ethereum transaction contains a cross-contract call or delegatecall, an access list transaction should be used.

## Using a fallback-extension as an implementation contract for upgradeable proxies

This pattern can be used with regular proxies. That is, a proxy contract can delegate to an implementation contract which then delegates to an extension. Keep in mind that tooling for upgrades (like Openzeppelin upgrade tools) are not designed to work with the extension fallback pattern and might not catch upgrade related issues.

## Learn More with RareSkills

Please see our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp) to learn more.

*Originally Published December 28, 2023*
