# Uniswap v2 router code walkthrough

The Router contracts provide a user-facing smart contract for

- safely [minting and burning LP tokens](https://www.rareskills.io/post/uniswap-v2-mint-and-burn) (adding and removing liquidity)
- safely [swapping pair tokens](https://www.rareskills.io/post/uniswap-v2-swap-function)
- They add the ability to swap Ether by integrating with the wrapped Ether (WETH) ERC20 contract.
- They add the [slippage](https://www.rareskills.io/post/uniswap-v2-price-impact) related safety checks omitted from the core contract.
- They add support for fee on transfer tokens.

## Router02 is everything Router01 does with support added for fee on transfer tokens

When we first open up the contracts folder in the periphery repository, we see three contracts

![uniswap v2 router github screenshot](https://static.wixstatic.com/media/935a00_b8eae8863df64996bd58019ea5d2bb54~mv2.jpg/v1/fill/w_740,h_494,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b8eae8863df64996bd58019ea5d2bb54~mv2.jpg)

Router02 is Router01 with additional functions for fee on transfer tokens. When we look at the interface of Router02, we can see it inherits from Router01 (<span style="color: red;">red box</span>) (which means it implements all of its functions), and has the following additional functions, which are all for doing operations with support for fee on transfer tokens (<span style="color: #c1c146;">yellow highlight</span>).

![uniswap v2 router inheritance](https://static.wixstatic.com/media/935a00_5ccf4b1e789c4a8886db540f33ba94b0~mv2.jpg/v1/fill/w_740,h_1186,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_5ccf4b1e789c4a8886db540f33ba94b0~mv2.jpg)

## swapExactTokensForTokens and swapTokensForExactTokens

Let's start with the Router functions for swapping tokens. There are two functions that accomplish this (<span style="color: green;">highlighted in green</span>).

![uniswap v2 router swap](https://static.wixstatic.com/media/935a00_3820d0c9f3534f9f9138da0f0afcebc7~mv2.jpg/v1/fill/w_740,h_574,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_3820d0c9f3534f9f9138da0f0afcebc7~mv2.jpg)

The difference in these function names is as follows:

- In `swapExactTokensForTokens` the "first token is exact" means that the amount of the input token you are swapping is a fixed quantity.
- In `swapTokensForExactTokens`, the "second token is exact" indicates that the amount of the output token you want to receive is a fixed quantity.
    
If a user is only swapping two tokens, then they will supply to these functions an `address[] calldata path` array (<span style="color: #008aff;">highlighted in blue</span>) `[address(tokenIn), address(tokenOut)]`. If they are hopping across pools, they will specify `[address(tokenIn), address(intermediateToken), …, address(tokenOut)]`.

### swapExactTokensForTokens

In the case of swap**Exact**TokensForTokens, the user specifies exactly how much of the first token they are going to deposit and the minimum amount of the output token they will accept.

For example, suppose we want to trade 25 `token0` for 50 `token1`. If this is the exact price at the current state, this leaves no tolerance for the price changing before our transaction is confirmed, leading to a revert. So we instead specify the minimum out to be 49.5 `token1`, implicitly leaving a 1% tolerance.

### swapTokensForExactTokens

In this case we specify we want exactly 50 `token1`, but we are willing to trade up to 25.5 `token0` to obtain in.

### Which swap function to use?

Most users using an EOA would probably opt to use the exact input function, because they need to have an approval step, and the trade would fail if they needed to input more than they approved. By having an exact input, they can approve the exact amount. Smart contracts integrating with Uniswap however may have more complex requirements, so the router gives them the option for both.

### How swap works

When the input is exact (swapExactTokensForTokens), the function predicts the expected output across a single swap or a chain of swaps. If the resulting output is below the user specified amount, the function reverts. Vice versa for exact output: it calculates the required input and reverts if it is above the user specified threshold.

Then both functions will transfer the user's tokens to the pair (remember, Uniswap V2 Pair requires the tokens to be sent into the contract before the pair contract function `swap()` is called). Finally, they both call the internal `_swap()` function discussed next.

![uniswap v2 router swap exact](https://static.wixstatic.com/media/935a00_f7f8ba0e33bf4f7e867b5a72a8d7f954~mv2.jpg/v1/fill/w_740,h_400,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_f7f8ba0e33bf4f7e867b5a72a8d7f954~mv2.jpg)

## The _swap() function

Under the hood, all publicly facing functions with the name `swap()` in the name call the `_swap()` internal function shown below.

Recall that the function signature for the core [swap function](https://www.rareskills.io/post/uniswap-v2-swap-function) specifies the `amountOut` for both tokens and the `amountIn` is implied by the amount that was transferred in before the function was called.

![uniswap v2 router internal swap function](https://static.wixstatic.com/media/935a00_631734d707bd424884a683c2729d3207~mv2.jpg/v1/fill/w_740,h_355,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_631734d707bd424884a683c2729d3207~mv2.jpg)

## _addLiquidity

Remember the safety checks for adding liquidity? Specifically, we want to make sure we deposit the two tokens at exactly the same ratio as what the pair currently has, otherwise the amount of LP tokens we mint is the worse of the two ratios between what we provide and what the pair balances are. However, the ratio could change between when the liquidity provider attempts to add liquidity and when the transaction is confirmed.

To guard against this, a liquidity provider must provide (as a parameter), the minimum balance they are seeking to deposit for token0 and token1 (UniswapV2 calls those `amountAMin` and `amountBMin`). Then they transfer in an amount higher than those minimums (UnsiwapV2 calls those `amountADesired` and `amountBDesired`). If the pair ratio has shifted in such a way that the minimums are no longer respected, then the transaction reverts.

_addLiquidity will take `amountADesired` and calculate the correct amount of tokenB that will respect the ratio. If this is amount is higher than `amountBDesired` (the amount of B the liquidity provider sent), then it will start with `amountBDesired` and calculate the optimal amount of B. The logic is show below. Note that adding liquidity may create a new pair contract if it doesn't already exist.

![uniswap v2 router add liquidity internal function](https://static.wixstatic.com/media/935a00_96d1b5f873b04ec3bf07b11c45d7d0b4~mv2.jpg/v1/fill/w_740,h_577,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_96d1b5f873b04ec3bf07b11c45d7d0b4~mv2.jpg)

For example, suppose the current pair balance is 100 token0 and 300 token1. We want to add 20 and 60 token0 and token1 respectively, but the pair ratio might change. So we instead approve the router for 21 token0 and 63 token1 while saying the minimum we want to deposit is 20 and 60. If the ratio shifts such that the optimal amount of token0 to deposit is 19.9, then the transaction reverts.


Recall that we said `quote` should not be used as an oracle, and that is still true. However for the purposes of adding liquidity we are not interested in the average of previous prices but the current price (pool ratio) now because the liquidity provider must respect it.

### addLiquidity(), and addLiquidityEth()

These functions should be self-explanatory. They first calculate the optimal ratio using `_addLiquidity` from above then transfer the assets to the pair, then call mint on the pair. The only difference is the addLiquidityEth function will wrap the Ether into WETH first.

![uniswap add liquidity with eth and weth](https://static.wixstatic.com/media/935a00_a817986edc7e440497a0fbd4032024fc~mv2.jpg/v1/fill/w_740,h_531,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_a817986edc7e440497a0fbd4032024fc~mv2.jpg)

## Removing Liquidity

Remove liquidity calls burn but uses parameters `amountAMin` and `amountBMin` (<span style="color:red">red highlights</span>) as safety checks to ensure that the liquidity provider gets back the amount of tokens they are expecting. If the ratio of tokens changes dramatically before the the liquidity tokens are burned, then the user burning the tokens won't get back the amount of token A or B that they are expecting.

The function `removeLiquidityEth` calls `removeLiquidity` (<span style="color: green;">green highlight</span>) but sets the router as the recipient of the tokens. The regular ERC20 token is then transferred to the liquidity provider, and the WETH is unwrapped to ETH, then sent back to the liquidity provider.

![uniswap v2 router remove liquidity](https://static.wixstatic.com/media/935a00_beaa1435518c4bbbbc38930b794d0237~mv2.jpg/v1/fill/w_740,h_510,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_beaa1435518c4bbbbc38930b794d0237~mv2.jpg)

### removeLiquidityWithPermit() and removeLiquidityETHWithPermit()

On line 109 in the file above with the <span style="color: gray;">gray comment</span> `send liquidity to pair`, this step assumes the pair contract has approval to transfer LP tokens from the liquidity provider to burn them. This means burning the LP tokens requires approving the pair first. This step can be skipped with `permit()`, since the LP tokens of Uniswap V2 is an [ERC20 Permit Token](https://eips.ethereum.org/EIPS/eip-2612). The function `removeLiquidityWithPermit()` receives a signature to approve and burn in one transaction. If one of the tokens is WETH, the liquidity provider would use `removeLiquidityETHWithPermit()`.

## Router02: supporting fee on transfer tokens

To handle fee on transfer tokens, the router cannot directly do it's calculations on arguments like `amountIn()` (for swap) or `liquidity()` (for removing liquidity). Adding liquidity is not affected by fee on transfer tokens because the user is only credited for what they actually transfer to the pair.

![uniswap v2 router supporting fee on transfer tokens](https://static.wixstatic.com/media/935a00_616774d489ad4441bfb48f65d4a53336~mv2.jpg/v1/fill/w_740,h_368,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_616774d489ad4441bfb48f65d4a53336~mv2.jpg)

  

![uniswap v2 supporting fee on transfer for removing liquidity](https://static.wixstatic.com/media/935a00_24162771cb5345529fec449bcc7e8b81~mv2.jpg/v1/fill/w_740,h_516,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_24162771cb5345529fec449bcc7e8b81~mv2.jpg)

## Wrappers around the UniswapV2Library

The rest of the functions in the Router library are wrappers around the [UniswapV2Library](https://www.rareskills.io/post/uniswap-v2-library) functions as shown below.

```solidity
    function quote(uint amountA, uint reserveA, uint reserveB) public pure override returns (uint amountB) {
        return UniswapV2Library.quote(amountA, reserveA, reserveB);
    }

    function getAmountOut(uint amountIn, uint reserveIn, uint reserveOut) public pure override returns (uint amountOut) {
        return UniswapV2Library.getAmountOut(amountIn, reserveIn, reserveOut);
    }

    function getAmountIn(uint amountOut, uint reserveIn, uint reserveOut) public pure override returns (uint amountIn) {
        return UniswapV2Library.getAmountOut(amountOut, reserveIn, reserveOut);
    }

    function getAmountsOut(uint amountIn, address[] memory path) public view override returns (uint[] memory amounts) {
        return UniswapV2Library.getAmountsOut(factory, amountIn, path);
    }

    function getAmountsIn(uint amountOut, address[] memory path) public view override returns (uint[] memory amounts) {
        return UniswapV2Library.getAmountsIn(factory, amountOut, path);
    }
}
```

## The deadline parameter

In the Uniswap V2 Routers, all the public functions have a deadline parameter. When you place a trade on Uniswap *right now*, it implies you want to trade at the current prices.

**When writing a smart contract that integrates with Uniswap, do not set the deadline to be `block.timestamp` or `block.timestamp` plus a constant.**

Your smart contract needs to *separately* ensure that the transaction submitted by the user is not too old. This means your own contract needs to accept a deadline parameter from the user and forward that to Uniswap or revert if `block.timestamp > deadline`.

### How to exploit old transactions

A malicious block builder can "hold on" to swap transactions and execute them much later when such transactions are useful for manipulating the price, or for dumping tokens on the user at an unfavorable price. A deadline parameter limits the time window where an attacker can conduct such an exploit. A deadline should be far enough in the future so that there is time to execute the transaction even during congestion, but not longer. This generally means the deadline should be on the order of minutes from when the transaction was signed.

However, if a smart contract doesn't incorporate a deadline or makes the parameter useless by ignoring the deadline and forwarding the current `block.timestamp` to Uniswap, then the user is not protected.

### Never set `amountMin` to zero or `amountMax` to `type(uint).max`

Another very common mistake is to set the amountMin to zero or amountMax to a very high value. This destroys the protection against price slippage and sandwich attacks.

## Conclusion

The Router contracts provide a user-facing mechanism for swapping tokens with slippage protection, possibly across multiple pools, and add support for trading ETH and fee-on-transfer tokens (in Router02). Depositing liquidity does not need to account for fee-on-transfer tokens because Uniswap only credits for what was actually transferred into the pool.

The depositing liquidity functions ensure the user only deposits at the exact ratio of the pool. Removing liquidity can be as simple as transferring LP tokens to the router then burning them, or include unwrapping WETH and withdrawing fee on transfer tokens.

Additionally, support for gas free approvals via ERC20 Permit are included.

A smart contract that integrates with Uniswap must not disable the protection against delayed swaps and price slippage.

*Originally Published Nov 10, 2023*
