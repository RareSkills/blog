# How Uniswap V2 computes the mintFee
Uniswap V2 was designed to collect 1/6th of the swap fees to the protocol. Since a swap fee is 0.3%, 1/6th of that is 0.05%, so 0.05% of every trade would go to the protocol.

Although this feature was never actually activated, we discuss this feature anyway since some forks may use it. It is also very easy to get the calculation wrong, so investing the time now to understand it will help you catch errors in similar calculations later.

## Prerequisites
You need to be familiar with all the prior chapters in the [Uniswap V2 Book](https://www.notion.so/How-Uniswap-V2-computes-the-mintFee-4f8cc0a84bc84254a4370fb2ccf5edb3?pvs=21) to be able to follow along.

## Collecting protocol fees during swaps is inefficient
It would be inefficient to collect 0.05% of the fee on every trade because that would require additional token transfers. Transferring ERC20 tokens requires a storage update, so transferring to two addresses instead of one would be considerably more expensive.

Therefore, the fee is collected when a liquidity provider calls [burn or mint](https://www.rareskills.io/post/uniswap-v2-mint-and-burn). Since these operations are infrequent compared to [swapping tokens](swapping tokens), this will lead to gas savings. To collect the `mintFee`, the contract calculates the amount of fees collected since that last happened, and mints enough LP tokens to the beneficiary address such that the beneficiary is entitled to 1/6th of the fee.

## Terminology of fee and mintFee
To avoid confusion in terminology, we refer to "fee" as the 0.3% collected from traders during the swap, and the "mintFee" the 1/6th of the 0.3% fee. Yes, having fee in both terms is not great nomenclature, but that's what we have to work with.

Liquidity is the square root of the products of the token balances in the pool. The justification for this formula was discussed in the Uniswap V2 swap function article. Some literature refers to this as $\sqrt{k}$ where $k = xy$ and $x$ and $y$ are the token balances in the pool (the reserves of $x$ and $y$). We refer to liquidity with $\ell$ since it is shorter to write than $\sqrt{k}$.

## Computing the mintFee assumptions
For this to work, Uniswap V2 relies on the following two invariants:

- If `mint()` and `burn()` are not called, the liquidity of the pool can only increase.
- The increase in liquidity is purely due to fees (or donations).
By measuring the increase in liquidity since the last `mint()` or `burn()` transaction, the pool knows how much fees were collected.

These would be good [invariant tests](https://www.rareskills.io/post/invariant-testing-solidity) to write, but for now we will take for granted that they are true.

## Example calculation of mintFee
Suppose at $t_1$ the pool starts at 10 `token0` and 10 `token1`.

After a lot of trading and fee collection, the new pool balance is 40 `token0` and 40 `token1` at $t_2$.

Liquidity is measured as the square root of the product of the two tokens, i.e. $\ell = \sqrt{k}$. The liquidity was 10 at $t_1$ and 40 at $t_2$. Said another way, $\ell_1 = 10$ and $\ell_2 = 40$. We are going to charge a fee on the growth from 10 to 40.

30 units of liquidity out of the total 40 at $t_2$ is due to swap fees.

We want to mint enough LP tokens, the "mintFee" such that the beneficiary receives 1/6th of the "fee portion" of the pool. That is, they should be entitled to 5 units of liquidity (30 / 6) that came from profit.

The protocol should not collect fees on any of the "original liquidiy" i.e. $\ell_1$. The protocol should only collect fees on the delta, i.e. $\ell_2 - \ell_1$.

When `mint()` or `burn()` is called, Uniswap mints LP tokens to the protocol fee recipient. This causes a dilution such that the *current supply* of LP tokens can redeem the original liquidity plus 5/6ths of the "profit liquidity" (liquidity from swap fees).

## Deriving the mint fee formula
Let’s use the following notation:

- $s$ supply of LP tokens before the dilutive protocol fee LP tokens are minted.

- $\eta$ is the amount of LP tokens that will be minted to the protocol. It should be enough to redeem 1/6th of the profit liquidity.

- $\ell_1$ is the liquidity of the original deposit, i.e. liquidity the LPs provided.

- $\ell_2$ is the liquidity of the original deposits and the liquidity that results from swap fees.

- $d$ is the amount of liquidity owed to the LPs net of the protocol fee. That is, the LPs should be entitled to their original deposit and 5/6ths of the profit.

- $p$ is the amount of liquidity owed to the protocol. This is 1/6ths of $\ell_2 - \ell_1$.

To compute $\eta$ we observe the following invariant must be true:

$$
\frac{\eta}{p}=\frac{s}{d}
$$

In other words, the previous total supply, $s$ LP tokens can redeem the liquidity due to the LPs, and $\eta$ LP tokens can redeem the amount due to the protocol.

The graphic below solves for $\eta$ in terms of the change in liquidity.

![algebraic derivation of the protocol fee](https://static.wixstatic.com/media/935a00_b6460940fa6f4d3a8e9c8bfc2d5701e7~mv2.jpg/v1/fill/w_1480,h_1110,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b6460940fa6f4d3a8e9c8bfc2d5701e7~mv2.jpg)

## The `_mintFee()` code of Uniswap V2
With that derivation in mind, the bulk of the Uniswap V2 `_mintFee` function should be self-explanatory. Here are some changes in notation:
- current liquidity after fees $\ell_2$ is `rootK`
- previous liquidity $\ell_1$ is `kLast`
- the supply of LP tokens before dilution $s$ is `totalSupply`
- the function is state changing, it mints the mintFee inside the function rather than return the calculation of the mintFee (<span style="color:#008aff">blue highlight</span>)
- the fee can be switched on an off with the flag `feeOn` which we haven't discussed yet

![_mintFee() solidity code](https://static.wixstatic.com/media/935a00_dc3d8ea8db88403aadba4d2ee1c48d05~mv2.png/v1/fill/w_1480,h_908,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_dc3d8ea8db88403aadba4d2ee1c48d05~mv2.png)

We will dive into this function some more, but first we want to note where `kLast` gets updated.

### Where `kLast` gets updated
In the code above, kLast is not set unless `feeOn` is switched to `false`. It is set at the completion of mint and burn but not swap because we are interested in measuring the growth of fees due to swaps between liquidity deposit and withdrawal events. The place `kLast` is set is marked with a <span style="color:#c1c146">yellow box</pan>.

#### Mint function updating `kLast`
![Mint function from Uniswap with the fee switch highlighted](https://static.wixstatic.com/media/935a00_fd8c409236f4494b9e175a7bac3b6cd4~mv2.png/v1/fill/w_1480,h_794,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_fd8c409236f4494b9e175a7bac3b6cd4~mv2.png)

#### Burn function updating `kLast`
![Burn function from Uniswap with the fee switch highlighted](https://static.wixstatic.com/media/935a00_d75b0789f82249f79c4b42225747e368~mv2.png/v1/fill/w_1480,h_822,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_d75b0789f82249f79c4b42225747e368~mv2.png)

### `_mintFee` code conditions
Now that we understand how kLast is updated, we can fully explain the _mintFee function.
![_mintFee function highlighted at branching points](https://static.wixstatic.com/media/935a00_dc3d8ea8db88403aadba4d2ee1c48d05~mv2.png/v1/fill/w_1480,h_908,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_dc3d8ea8db88403aadba4d2ee1c48d05~mv2.png)

Let's consider the possibilities in the code snippet above, repeated for convenience.
1. The `feeOn` is `false`, nothing is minted (<span style="color:Green">green highlight</span>)
2. The `feeOn` is `false`, `kLast` is zero (<span style="color:#c1c146">yellow highlight</span>)
3. The `feeOn` is `false`, `kLast` is not zero (<span style="color:#c1c146">yellow highlight</span>)
4. The `feeOn` is `true`, but there was no growth in liquidity (<span style="color:Orange">orange highlight</span>)
5. The `feeOn` is `true`, and there was liquidity growth (<span style="color:Orange">orange highlight</span>), so the mint fee applies (<span style="color:#008aff">blue highlight</span>)

It's easier to see the logic in a decision tree, so here is the decision tree with the branches colored the same as the if statements.

![decision tree of _mintFee showing the 5 branches](https://static.wixstatic.com/media/935a00_cd23aafc2f7645428bc30d245244a0e2~mv2.jpg/v1/fill/w_1151,h_454,al_c,q_85,enc_auto/935a00_cd23aafc2f7645428bc30d245244a0e2~mv2.jpg)

## Learn more with RareSkills
Please see our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) to see our course offerings.

*Originally Published November 14, 2023*
