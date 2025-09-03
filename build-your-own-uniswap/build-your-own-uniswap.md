# Checklist for building a Uniswap V2 clone

It’s very educational to rebuild Uniswap v2 from scratch using modern Solidity (or Huff if you really want to do it in hard mode). Here are some pointers and suggestions for doing so.
- Use an updated version of Solidity. Be aware that this will lead to syntactic changes.
- Replace the fixed point numbers with custom data types.
- Use the [Solady](https://github.com/Vectorized/solady) ERC20 for the ERC20 to save gas.
- Don’t use Uniswap V2's current reentrancy protection, it isn’t gas efficient anymore, use OpenZeppelin’s or some other alternative.
- Be careful to add unchecked in the price oracle, it expects to overflow.
- Don’t use `safeMath` with Solidity 0.8.0 or higher.
- If you are not implementing the router separately, safety checks for slippage need to be built into the contract. EOAs cannot send tokens as part of the transaction.
- Be sure to put the reentrancy lock in the right places. Is Uniswap V2 subject to read-only reentrancy? Why or why not?
- The factory contract can be simplified without assembly because of Solidity updates.
- Watch out for fee-on-transfer tokens or re-basing tokens.
- The square root function can be done more efficiently with the Solady library, but be sure you are rounding in the correct direction.
- Uniswap hardcodes the fee with magic numbers, this isn’t an ideal way to write code.
- Don’t forget to follow the professional Solidity style guide (which Uniswap V2 does not follow).
- The way the factory tracks pairs isn’t gas efficient, try to improve it.
- Some storage variables in the original implementation could be made to be immutable (immutable variables were not available at the time Uniswap V2 was launched).
- Custom errors will lead to a cheaper deployment cost than require statements (generally)
- When burning the initial supply, make sure that the `totalSupply` does not go down to zero. Otherwise, the defense against the first deposit attack will not work. Some implementations of burn reduce the total supply rather than locking up the funds as Uniswap intends.
- Be careful to `burn`, `mint`, and `update` reserves in the correct order when computing shares.
- Uniswap V2’s `_safeTransfer` is subject to the memory expansion attack (very unlikely, but still should be guarded against). Since only one bool will be read, it’s best to `returndatacopy` only one word. Try to avoid getting into the habit of reading the entire return data from other contracts.
- A good invariant test is liquidity should never go down if `burn` is not called.
- Don’t use the balance variables or pool balance as an oracle, as these are vulnerable to flash loan attacks.
- Remember to always round in favor of the pool when doing trades or minting or burning shares.
- Don’t forget to write unit tests.
- Give our book on [Solidity gas optimization](https://www.rareskills.io/post/gas-optimization) a read for inspiration about how to improve performance (after you finish your second or third draft of the Uniswap clone and have written tests). If you want to do this in hard mode, there are several opportunities to improve the gas using assembly without substantial sacrifices in readability.
- Try out [Damn Vulnerable Defi Puppet V2](https://www.damnvulnerabledefi.xyz/challenges/puppet-v2/). It should be easy for you to solve at this point.

## Learn more with RareSkills

This material is part of our advanced [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp). Please see the program to learn more.

*Originally Published November 1, 2023*
