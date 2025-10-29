# Learn Solidity: Easy to Learn, Hard to Master?
## Is Solidity hard to learn?

[Learning solidity](https://www.rareskills.io/learn-solidity) as a language is arguably one of the easier languages to learn. However, learning the Ethereum environment is hard.

It looks very similar to Javascript, or pretty much any curly bracket language derived from C.

If statements, for loops, class inheritance, variable types, are all very familiar.

Solidity does have some oddities unique to moving cryptocurrency around. For example, each function call has an environment variable that indicates how much ether was sent with the function call and it has some specific APIs for interacting with other smart contracts. Solidity also has strange instructions like [delegatecall](https://www.rareskills.io/post/delegatecall) and selfdestruct which are not found other languages, but those are easy to grasp after mulling over the documentation for a bit.

However, Solidity and Ethereum development can be full of surprises. Here are just three examples.

##

## Seemingly minor changes can result in very large differences in gas cost

![Solidity code demonstrating efficient and inefficient functions](https://static.wixstatic.com/media/935a00_50edf803a62f4ead8010289c19823786~mv2.png/v1/fill/w_666,h_555,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_50edf803a62f4ead8010289c19823786~mv2.png)

What this contract does is very simple. It transfers tokens away from another contract to itself, then it transfers it to the caller in the same transaction.

However, openFaucetInefficient and openFaucetMoreEfficient can have wildly different gas costs. Why? Under the hood, ERC20 tokens store the user’s balance as a storage variable. When a storage variable goes from zero to non-zero, this causes the variable to be created on the blockchain. And the creation step has a high cost associated with it. When a variable is set to zero, it is implicitly erased. So the first function is repeatedly creating and erasing a storage variable.

The second function is much more efficient. It creates the storage variable for its own balance, then ensures it doesn’t get destroyed until the end when it transfers away the final token. This prevents unnecessary creation of storage variables.

The third function, which has a bizarre for-loop construction is even more efficient. Because of quirks in the Solidity compiler, re-arranging a for loop in this manner is more efficient, even when you tell the compiler to run automatic efficiency improvements on your code.

How would you know this? Well, there is no straightforward way to know. That's why Solidity is not easy to master.

## I/O operations can be undone

![Solidity code showing revert after I/O write](https://static.wixstatic.com/media/935a00_393e8f2679bc4f859b5e4c3f026ac6e6~mv2.png/v1/fill/w_666,h_550,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_393e8f2679bc4f859b5e4c3f026ac6e6~mv2.png)

Both x and y are storage variables, and you can think of them as being stored on persistent disk. They keep their values between transactions.

When quizzed, many developers assume that x will be set to the newValue and y will be set to 2 times x if x is less than or equal to 10. For example, if newValue is 5, x will be 5 and y will be 10. If newValue is 20, x will be 20 and newValue will not be changed.

But this isn’t what happens. If a revert happens anywhere in the transaction, the write to X gets undone.

This is very counter-intuitive for developers, because most I/O operations don’t revert. However, all transactions on Ethereum are atomic. So x can never be greater than 10, and x and y must change in tandem.

## Innocuous functions can lead to re-entrancy attacks

![Solidity code with re-entrancy vulnerability](https://static.wixstatic.com/media/935a00_92f9abb91e964e36b593785942662bb4~mv2.png/v1/fill/w_666,h_322,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_92f9abb91e964e36b593785942662bb4~mv2.png)

The code above appears to send people an ERC20 token and an ERC1155 token when they call mintTokens. Ostensibly, each address can only mint one time because of the alreadyClaimed check.

However, it is possible to drain the contract of all the tokens in a single transaction. The \_mint function in ERC1155 doesn’t just mint a token, it also hands control over to the sender of the transaction — before it has updated that the sender has claimed their tokens. This allows the sender to claim all the tokens for themselves by recursively calling mintTokens when \_mint hands control back to them.

There is no rhyme or reason to which functions hand control over to other contracts. One simply has to memorize them.

## Conclusion

As you can see above, a seemingly simple language can be full of surprises. We’ve only scratched the surface here. Did you know it’s possible for “immutable” smart contracts to change their bytecode? How about bad code designs that allow buyers to double-spend their ether without a re-entrancy attack? Can you generate random numbers securely despite the blockchain being fully deterministic and transparent?

Blockchain is full of unknown unknowns despite using a programming language that is “easy” to learn. This is why hacks are so common.

Solidity can be grasped in a weekend. Here is our free tutorial to [Learn Solidity](https://www.rareskills.io/learn-solidity) quickly if you already know another programming language.

But mastering the ecosystem does not happen in a matter of days.

Do you want to master the ecosystem? Apply now to our fully remote [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp).

*Originally published Nov 8, 2022*
