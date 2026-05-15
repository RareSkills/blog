# UniswapV2Library Code Walkthrough

## UniswapV2Library

The [Uniswap V2 Library](https://github.com/Uniswap/v2-periphery/blob/master/contracts/libraries/UniswapV2Library.sol) simplifies some interactions with pair contracts and is used heavily by the Router contracts. It contains eight functions that are not state-changing,. They are also handy for integrating Uniswap V2 from a smart contract.

## getAmountOut() and getAmountIn()

If we want to predict the amount of `token y` we will get if we supply a fixed amount of `token x`, then we can derive the amount out using the sequence below (ignoring fees for simplicity). Let `x` be the incoming token, `y` be the outgoing token, `Δx` be the amount coming in and `Δy` be the amount going out

$$
\begin{align*}
    xy &= (x + \Delta x)(y - \Delta y) \\
    y - \Delta y &= \frac{xy}{x + \Delta x} \\
    -\Delta y &= \frac{xy - y(x + \Delta x)}{x + \Delta x} \\
    -\Delta y &= \frac{-y \Delta x}{x + \Delta x} \\
    \Delta y &= \frac{y \Delta x}{x + \Delta x}
\end{align*}
$$

$$
\text{amount\_out} =
\frac{
  \text{reserve\_out} \cdot \text{amount\_in}
}{
  \text{reserve\_in} + \text{amount\_in}
}
$$

With that in mind, the function for `getAmountOut()` in the `UniswapV2Library.sol` should be self-explanatory. Note that the numbers are scaled by 1,000 to account for the 0.3% fee. The derivation for `getAmountIn()` with fees is an exercise for the reader.

![functions getAmountOut and getAmountIn](https://static.wixstatic.com/media/935a00_b9765b82621d404db1c740090b34e402~mv2.jpg/v1/fill/w_1480,h_558,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b9765b82621d404db1c740090b34e402~mv2.jpg)

## Chaining getAmountOut() to getAmountsOut() for pair hops

### getAmountsOut() and getAmountsIn()

If a trader supplies a sequence of pairs, (A, B), (B, C), (C, D) and iteratively calls `getAmountOut` starting with a certain amount of `A`, then the amount of token `D` that will be received can be predicted

The address of the UniswapV2 pair contract for each (A, B), (B, C), etc. is deterministically derived from the addresses of the tokens and the address of the factory that deployed the pair using the `create2` function. Given two tokens (A, B) and a factory `address`, `pairFor()` derives the address of the pair UniswapV2 pool contract for that pair, using `sortTokens()` as a helper function.

![functions pairFor and sortTokens](https://static.wixstatic.com/media/935a00_a3ac066c3bca4fad88be17b06fa5d2f0~mv2.jpeg/v1/fill/w_1480,h_544,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_a3ac066c3bca4fad88be17b06fa5d2f0~mv2.jpeg)

Now that we know the addresses for all the pairs, we can get the reserves of each one and predict how much tokens we will receive at the end of the chain of swaps. Below is the code for `getAmountsOut()` (emphasis on “Amounts” instead of “Amount”). The function `getAmountsIn()` simply does the same thing in reverse so we won’t show it here.

Note a couple things:
- The smart contract doesn’t figure out the optimal sequence of pairs on its own, it needs to be told the list of pairs to calculate the chain of swaps over. This is best done off-chain.
- It doesn’t just return the final token `amountOut` in the chain, but the amount out at every step.

![getAmountsOut function](https://static.wixstatic.com/media/935a00_6e4f70d756374b0f9b19fcce962d531c~mv2.jpg/v1/fill/w_1480,h_280,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_6e4f70d756374b0f9b19fcce962d531c~mv2.jpg)

## getReserves()

The function `getReserves` is simply a wrapper around the function `getReserves` from the Uniswap V2 pair contract except that it also removes the timestamp when the price was last updated. The function for `getReserves` from the pair contract is also shown for easy comparison (<span style="color: purple">purple comments</span>).

![uniswapV2Library getreserve](https://static.wixstatic.com/media/935a00_4446a5b0318941fe86ba7b7fcb3c9ee8~mv2.jpg/v1/fill/w_1480,h_200,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_4446a5b0318941fe86ba7b7fcb3c9ee8~mv2.jpg)

Core function:

![core v2 getReserves](https://static.wixstatic.com/media/935a00_61052d0a20bf4bb78c74521ba7ef79a4~mv2.jpg/v1/fill/w_1480,h_182,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_61052d0a20bf4bb78c74521ba7ef79a4~mv2.jpg)

## quote()

Recall that the price of an asset follows the following formula:

$$
\mathsf{price}(\text{foo}) = \frac{\mathsf{reserve(\text{bar})}}{\mathsf{reserve}(\text{foo})}
$$

This function returns the price of foo denominated in bar as of the last update. **This function should be used with care as it is vulnerable to flash loan attacks.**

![uniswap v2 library quote function](https://static.wixstatic.com/media/935a00_39060f4174ea43f4be9f5a46043aff2c~mv2.jpg/v1/fill/w_1480,h_210,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_39060f4174ea43f4be9f5a46043aff2c~mv2.jpg)

## Using UniswapV2Library

If you want to predict how much to put into or expect out of a trade, or a sequence of trades across pairs, the `UniswapV2Library` is the tool to use.

## Practice

Try out [DamnVulnerableDefi Puppet V2](https://www.damnvulnerabledefi.xyz/challenges/puppet-v2/). It should be easy for you to identify the security issue at this point.

## Learn more with RareSkills

This material is part of our advanced [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp). Please see the program to learn more.

*Originally Published November 6, 2023*