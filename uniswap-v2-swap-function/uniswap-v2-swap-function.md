# Breaking Down the Uniswap V2 Swap Function

Uniswap V2’s swap function is cleverly designed, but many devs find its logic counterintuitive the first time they encounter it. This article explains how it works in depth.

Here is the code reproduced below:

![uniswap v2 swap function](https://static.wixstatic.com/media/935a00_3c6633164b96444ebf6861c42e8dbb5b~mv2.png/v1/fill/w_740,h_461,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_3c6633164b96444ebf6861c42e8dbb5b~mv2.png)

Admittedly, this is a wall of code, but let’s break it down.

-   On line 170-171 (indicated with a <span style="color:#c1c146">yellow box</span>), the function directly transfers out the amount of tokens that the trader requested in the function arguments. **There is no place inside the function where tokens are transferred in.** Scan the code and see if you can find where the tokens are transferred in, it doesn’t exist. But this does not mean we can just call swap and drain all the tokens we want to!

-   The reason we can remove tokens right away is so that we can do flash loans. Of course, the require statement on line 182 (<span style="color:Orange">orange arrow</span>) will require us to pay back the flash loan with interest.

-   At the top of the function, there is a comment which says the function should be called from another smart contract which implements important safety checks. That means **this function in particular is missing safety checks** (red underline). We’ll want to determine what those are.

-   The **variables _reserve0 and _reserve1** (<span style="color:#008aff">blue underline</span>) are read on lines 161, 176-177, and 182, but they **are not written to in this function**.

-   Line 182 (<span style="color:Orange">orange arrow</span>) does not strictly check if X × Y = K. It checks if balance1Adjusted × balance2Adjusted ≥ K. **This is the only require statement that does something “interesting.”** The other require statements check that values aren’t zero or that you aren’t sending the tokens to their own contract address.

-   balance0 and balance1 are directly read from the actual balance of the pair contract using ERC20 balanceOf

-   Line 172 (below the <span style="color:#c1c146">yellow box</span>) is only executed if data is non-empty, otherwise it is not executed

Using these observations, we will make sense of this function one feature at a time.

## Flash Borrowing

Users do not have to use the swap function for trading tokens, it can be used purely as a flash loan.

![uniswap v2 flash borrowing](https://static.wixstatic.com/media/935a00_a54daf3d2a764e2d83cf4f2db18c6c1e~mv2.png/v1/fill/w_740,h_460,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_a54daf3d2a764e2d83cf4f2db18c6c1e~mv2.png)

The borrowing contract simply requests the amount of tokens they wish to borrow <span style="color:#8d28a4">(A)</span> without collateral and they will be transferred to the contract <span style="color:#8d28a4">(B)</span>.  

The data that should be provided with the function call is passed in as a function argument <span style="color:#8d28a4">\(C\)</span>, and this will be passed to a function that implements

IUniswapV2Callee. The function uniswapV2Call must pay back the flash loan plus the fee or the transaction will revert.

### Swap requires using a smart contract

If a flash loan is not used, the incoming tokens must be sent as part of calling the swap function.

It should be clear that **only a smart contract is able to interact with a swap function**, because an EOA cannot simultaneously send the incoming ERC20 tokens and call swap in one transaction without the aid of another smart contract.

## Measuring the amount of incoming tokens

The way Uniswap V2 “measures” the amount of tokens sent in is done on line 176 and 177, marked with the <span style="color:#c1c146">yellow box</span> below.

![swap measure reserves and balances](https://static.wixstatic.com/media/935a00_0b98b9ec5b0f46318c83353606b54f32~mv2.png/v1/fill/w_740,h_324,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_0b98b9ec5b0f46318c83353606b54f32~mv2.png)

Remember, _reserve0 and _reserve1 are not updated inside this function. They reflect the balance of the contract before the new set of tokens were sent in as part of the swap.

One of two things can happen for each of the two tokens in the pair:

1.  The pool had a net increase in the amount of a particular token.
    
2.  The pool had a net decrease (or no change) in the amount of a particular token.

The way the code determines which situation happened with the following logic:

```solidity
currentContractbalanceX > _reserveX - _amountXOut

// alternatively

currentContractBalanceX > previousContractBalanceX - _amountXOut
```

If it measures a net decrease, the ternary operator returns zero, otherwise it will measure the net gain of tokens in.

```solidity
amountXIn = balanceX - (_reserveX - amountXOut)
```

It is always the case that _reserveX > amountXOut because of the require statement on line 162.

![An image of the require statement](https://static.wixstatic.com/media/935a00_66290d79a547494282218b32b751005c~mv2.png/v1/fill/w_740,h_19,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_66290d79a547494282218b32b751005c~mv2.png)

Some examples.

-   Suppose our previous balance was 10, amountOut is zero, and currentBalance is 12. That means the user deposited 2 tokens. amountXIn will be 2.

-   Suppose our previous balance was 10, amountOut is 7, and currentBalance is 3. amountXIn will be 0.

-   Suppose our previous balance was 10, amountOut is 7, and currentBalance is 2. amountXIn will still be zero, not -1. It is true that the pool had a net loss of 8 tokens, but amountXIn cannot be negative.

-   Suppose our previous balance was 10, and amountOut is 6. If the currentBalance is 18, then the user “borrowed” 6 tokens but paid back 8 tokens.

**Conclusion: amount0In and amount1In will reflect the net gain if there was a net gain for the token, and they will be zero if there was a net loss of that token.**

## Balancing XY = K

Now that we know how many tokens the user sent in, let’s see how to enforce XY = K.

The code again is

![An image of code](https://static.wixstatic.com/media/935a00_b7e8d0adcb204f4b9d41a0a297aab8b0~mv2.png/v1/fill/w_740,h_47,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b7e8d0adcb204f4b9d41a0a297aab8b0~mv2.png)

Uniswap V2 charges a hardcoded 0.3% per swap, which is why we see the numbers 1000 and 3 at play, but lets simplify this by changing it to the case where Uniswap V2 charged no fees. This means we can remove the .sub(amountXIn.mul(3)) term and not multiply by 1000 on lines 180 to 181 or 1000**2 on line 182.

The new code would be

```
require(balance0 * balance1 >= reserve0 * reserve1, "K");
```

This is saying

$$
\begin{align*}
X_\text{new}Y_\text{new} &\geq X_\text{prev}Y_\text{prev}\\
K_\text{new}&\geq_\text{prev}
\end{align*}
$$

### K is not really constant

It’s a bit misleading to say “K remains constant” even though the AMM formula is sometimes referred to as a “constant product formula.”

Think about it this way, if someone donated tokens to the pool and changed the value of K, we wouldn’t want to stop them because they made us liquidity providers richer, right?

Uniswap V2 doesn’t prevent you from “paying too much” i.e. transferring in too many tokens in during the swap (this is related to one of the safety checks, which we will get to later).

We would be upset if there was a net loss in the pool, which is what the require statement is checking. If K gets larger, it means the pool got larger, and as liquidity providers, that’s what we want.

### Accounting for fees

But not only do we want K to get larger, we want it to get larger by at least an amount that enforces the 0.3% fee.

Specifically, the 0.3% fee applies to the size of our trade, not the size of the pool. It only applies to the tokens that go in, not on the tokens that go out. Some examples:

-   Suppose we put in 1000 of token0 and remove 1000 of token1. We would need to pay a fee of 3 on token0 and no fee on token1.
    
-   Suppose we borrow 1000 of token0 and do not borrow token1. We are going to have to put 1000 of token0 back in, and we will have to pay a 0.3% fee on that — 3 of token0.

Observe that if we flash borrow one of the tokens, it results in the same fee as swapping that token for the same amount. You pay fees on tokens in, not on tokens out. But if you don’t put tokens in, there is no way for you to borrow or swap.

Remember, reserve0 and reserve1 represent the old balances, and balance0 and balance1 represent the updated balances.

With that in mind, let’s write the code below should be self-explanatory. The multiplying by 1000 and 3 is to simply accomplish “fractional” multiplication since it cancels out in the end.

![An image of fractional multiplication](https://static.wixstatic.com/media/935a00_2f3123e5591b44359ad9eb70393e71e6~mv2.png/v1/fill/w_740,h_47,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_2f3123e5591b44359ad9eb70393e71e6~mv2.png)

The code is accomplishing the following formula:

$$
\begin{aligned}
(\text{new\_balance}_0 - 0.003 \times \text{amountIn}_0)
&\times
(\text{new\_balance}_1 - 0.003 \times \text{amountIn}_1)
\\
&\geq
(\text{prev\_balance}_0 \times \text{prev\_balance}_1)
\end{aligned}
$$

That is, the new balance must increase by 0.3% of the amount in. In the code, the formula is scaled by multiplying each term by 1,000 because Solidity doesn’t have floating point numbers, but the math formula shows what the code is trying to accomplish.

## Updating Reserves

Now that the trade is completed, then the “previous balance” must be replaced with the current balance. This happens in the call to the _update() function at the end of swap().

![update reserves function call](https://static.wixstatic.com/media/935a00_5193907a21a44de0bb29396c6f9e51ca~mv2.png/v1/fill/w_740,h_454,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_5193907a21a44de0bb29396c6f9e51ca~mv2.png)

### The _update() function

![update reserves in _update function](https://static.wixstatic.com/media/935a00_02048778d249465daee1f467f739199a~mv2.png/v1/fill/w_740,h_277,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_02048778d249465daee1f467f739199a~mv2.png)

There is a lot of logic here to handle the TWAP oracle, but all we care about for now is lines 82 an 83 where the storage variables reserve0 and reserve1 are updated to reflect the changed balances. The arguments _reserve0 and _reserve1 are used to update the oracle, but they are not stored.

## Safety Checks

There are two things that can go wrong:

1.  The amountIn is not enforce to be optimal, so the user might overpay for the swap
    
2.  AmountOut has no flexibility as it is supplied as a parameter argument. If the amountIn turns out to not be sufficient relative to amountOut, the transaction will revert and gas will be wasted.
    

These circumstances can happen if someone frontruns a transaction (intentionally or not) and changes the ratio of assets in the pool in an undesirable direction.

## Learn more with RareSkills

This article is part of our advanced [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp). Please see the curriculum to learn more.

*Originally Published October 28, 2023*
