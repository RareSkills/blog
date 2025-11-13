# Positions in Uniswap v3

Adding liquidity to an AMM means depositing tokens into the AMM pool. Liquidity providers do this in the hope of earning fees from users who swap with that pool.

In Uniswap v2, when an LP adds liquidity, they receive shares of the pool in the form of Liquidity Provider tokens (LP tokens), representing the percentage of tokens in the pool to which they are entitled, including fees. These LP tokens are fungible ERC-20 tokens.

In Uniswap v3, this approach doesn't work because the LP chooses the range where they want to deposit liquidity—the lower and upper ticks. Therefore, the protocol needs to track each deposit individually in a non-fungible manner. This brings us to the concept of **positions**.

> **When an LP adds liquidity in a range, we say they open or modify a position**. It is *opened* when the position does not yet exist, and *modified* when it does.
> 

A range consists of two ticks — a lower and an upper tick. Depositing liquidity into a range means increasing the real reserves within that range. This is achieved using the `mint` function in [`UniswapV3Pool.sol`](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L457), whose interface is shown below.

```solidity
// UniswapV3Pool.sol

function mint(
  address recipient, // the owner of the position
  int24 tickLower,
  int24 tickUpper,
  uint128 amount, // amount in liquidity
  bytes calldata data // will be explained in a future chapter, it is not necessary for our discussion
) external override lock returns (uint256 amount0, uint256 amount1) {
```

The name `mint` for this function is reminiscent of Uniswap v2, where adding liquidity minted ERC-20 tokens for the liquidity provider. Although this no longer occurs in v3, the name has been preserved - this time to refer to minting a position.

The goal of this chapter is to examine how and where the protocol stores information about these positions.

## The `positions` mapping

Positions are stored in a mapping called `positions`, located in the `UniswapV3Pool` contract, as shown in the image below. 

![positions_mapping.png](https://r2media.rareskills.io/UniswapV3Positions/image7.png)

**The key that identifies a position is formed by the Keccak hash of the position owner's address, the lower tick, and the upper tick.** 

Thus, if this is the first time liquidity is deposited between ticks -10 and 10 for owner `0xA`, a position will be created with the key `keccak(0xA, -10, 10)`.

Below is a Solidity code used to generate the key for a position.

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Position {

    function getKey(
        address owner,
        int24 tickLower, 
        int24 tickUpper) 
    public pure returns (bytes32 key) {
        key = keccak256(abi.encodePacked(owner, tickLower, tickUpper));
    }
    
}
```

Once created, if more liquidity is deposited (or withdrawn) between ticks -10 and 10 for owner `0xA`, the position will be modified, since it already exists.

The mapping value is of type `Position.Info`, which is a struct named `Info` located in the `Position` library within the [Position.sol](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Position.sol) contract.

This struct, shown below, stores the position's `liquidity` (orange box), as well as four other fields related to fees and tokens owned by the position (green box). We will postpone the discussion of these other fields until we cover fees and liquidity withdrawal from a position.

![positions1.png](https://r2media.rareskills.io/UniswapV3Positions/image6.png)

For now, we can think of it this way: if liquidity is added within a range by a given owner, a position is created. After that, this position can be modified by adding or removing liquidity.

## Positions are non-fungible

In Uniswap v2, when an LP provides liquidity, ERC-20 LP tokens are minted for the LP. These LP tokens are fungible and represent a share of the pool’s assets.  This means that, if two LPs each own 1000 LP tokens and use their LP tokens to redeem their shares in the pool, the asset tokens they receive will be the same.

In Uniswap v3, positions are non-fungible. This means that if two LPs have different positions and use these positions to redeem their shares in the pool, they will probably not receive the same amount of assets, as they likely don’t represent the same range of ticks or the real reserves the LPs contributed between those ticks.

It is possible to handle these non-fungible positions as ERC-721 non-fungible tokens through a peripheral contract, but the core contract itself does not do this—it is not an ERC-721 contract and does not allow transferring positions to third parties. The core contract only allows opening and modifying positions.

## Fees in Uniswap v3

Fees in Uniswap v3 also work very differently from those in Uniswap v2. In Uniswap v2, whenever a swap occurs, the fees are added to the pool, increasing the amount of tokens each share is entitled to.

In Uniswap v3, a position is only entitled to fees from swaps that occurred—partially or entirely—within its tick range. Therefore, if a LP opens a position that is never involved in a swap, that position earns no fees. This incentivizes LPs to open positions in regions where swaps are likely to happen, thereby increasing liquidity where it is most needed.

Since fees are no longer shared proportionally among all LPs but are attributed individually to each position, they must be tracked separately. 

How the protocol calculates how many of the fees a position is entitled to is not straightforward and involves a combination of several variables, including the `feeGrowthInside0LastX128` and `feeGrowthInside1LastX128` variables present in the Position's `Info` struct.

As a high-level explanation, the protocol keeps track of the fees accumulated in the pool since its creation, as well as how many fees were collected below and above each tick that serves as a position's boundary.

This allows the protocol to calculate how much of the accumulated fees were collected between the lower and upper ticks of a position and use the position's `feeGrowthInside0LastX128` and `feeGrowthInside1LastX128` variables to determine how much of these fees belong to that particular position.

The details of how the protocol achieves this will be covered in future chapters.

## A pool is made up of several positions

We've been saying that a pool consists of segments, so we need to connect the idea of segments and positions. They are not the same, since positions can overlap. Let's explore this through an example.

Consider only two positions: 

1. Between ticks -10 and 5 with liquidity of 200, shown in the red box below. 
2. Between ticks 0 and 10 with liquidity of 100, illustrated in the blue box below. 

![A diagram showing how positions relate to liquidity levels between ticks](https://r2media.rareskills.io/UniswapV3Positions/image4.png)

- Between ticks -10 and 0, there is only one position with liquidity of 200, so the liquidity for this segment will be 200.
- Between ticks 0 and 5, two positions overlap: one with liquidity of 200 and the other with liquidity of 100, so the liquidity for this segment will be 300.
- Between ticks 5 and 15, there is only one position with liquidity of 100, so the liquidity for this segment will be 100.

The segments are shown in the figure above on the right.

Below is an interactive tool where the reader can create multiple positions, and the tool automatically calculates segments based on these positions. It is also possible to change the current price and view the liquidity of the segment in which the current price is located.

<iframe
  src="https://rareskills.io/uniswap-v3-positions-tool"
  width="900"
  height="900"
  frameborder="0">
</iframe>

However, the protocol does NOT compute all the segments based on the positions, as we just did. This would be highly inefficient, since the protocol does not need such a global picture.

In a later chapter, we will discuss how the protocol efficiently handles positions to compute segments. For now, we temporarily treat that computation as a black box.

In the next section, we will take a closer look at the `mint` function and what the LP must deposit to open a position.

## The `mint` function

To add liquidity to a pool, it is necessary to deposit tokens. As we have seen, this is done through the `mint` function, whose interface is shown again below.

```solidity
function mint(
  address recipient, // the owner of the position
  int24 tickLower,
  int24 tickUpper,
  uint128 amount, // amount in liquidity
  bytes calldata data // will be explained in a future chapter, it is not necessary for our discussion
) external override lock returns (uint256 amount0, uint256 amount1) {
```

This function expects five parameters:

- The position owner's address (`recipient`), the lower tick (`tickLower`), and the upper tick (`tickUpper`) of the position.
- The `data` parameter, of type `bytes`, will be explained later and is not important for the current discussion.
- The `amount` parameter, which is the **amount of liquidity** the LP wants to add to the pool between the lower and upper tick.

*Note that `amount` is of type `uint128`, which means it cannot be negative. The `mint` function is used only to add liquidity, not to remove it—removals are handled by the `burn` function, which will be discussed later.*

The amount of tokens that need to be deposited to add this `amount` of liquidity between the lower and upper ticks must then be calculated by the `mint` function.

This is what we will see next.

## Tokens required to open a position

The amount of tokens corresponding to liquidity $L$ between a lower and an upper tick is the real reserves of the segment corresponding to that position, i.e., a segment with liquidity $L$ between the lower and upper tick.

We've already learned that a segment's real reserves depend not only on the tick boundaries and liquidity $L$ but also on the current price. The rule is as follows:

1. When the current price is at or above the upper tick, the segment has real reserves only in tokens Y.
2. When the current price is at or below the lower tick, the segment has real reserves only in tokens X.
3. When the current price is between the lower and upper ticks, the segment has real reserves in both tokens X and Y.

These three scenarios are illustrated in the figure below, where the red ray represents the current price $p$, $p_l$ represents the lower tick and $p_u$ represents the upper tick.

![A diagram showing how when the price is higher than the upper tick of the range, the reserves are entirely token y and when the price is below the range, the reserves are all token x. When the price is in between, both tokens make up the liquidity in the range.](https://r2media.rareskills.io/UniswapV3Positions/image3.png)

If the LP wants to add liquidity $L$ between the lower tick $p_l$ and the upper tick $p_u$, the protocol calculates  $x_r$ (real reserves in tokens X) and $y_r$ (real reserves in tokens Y) according to the three scenarios described above.

1. For scenario 1, the real reserves in tokens Y are

$$
y_r = L\sqrt{p_u} - L\sqrt{p_l}
$$

1. For scenario 2, the real reserves in tokens X are

$$
x_r = \frac{L}{\sqrt{p_l}} - \frac{L}{\sqrt{p_u}}
$$

1. For scenario 3, the real reserves in tokens X and Y are

$$
\begin{align*}
x_r &= \frac{L}{\sqrt{p}} - \frac{L}{\sqrt{p_u}} \\
y_r &= L\sqrt{p} - L\sqrt{p_l}
\end{align*}
$$

These token amounts required to add liquidity $L$ to the position are calculated and returned by the `mint` function, as shown in the red box below.

![The return signature of mint](https://r2media.rareskills.io/UniswapV3Positions/image8.png)

However, for the end user, liquidity can be a highly abstract concept. It is much more common for the end user to think in terms of tokens rather than liquidity—for example, a user might want to provide liquidity by depositing 100 tokens X without having any idea of the amount of liquidity that 100 tokens X represents.

Adding liquidity by choosing token amounts can be facilitated through an intermediary contract that acts as a bridge between the end user and the core contract and calculates the conversion between token amounts and liquidity.

## Opening a position through a position manager

The `mint` function in the core contract is meant to be called by another contract and NOT by EOAs.

When the `mint` function is called, it calls back the address that triggered it through the `uniswapV3MintCallback` function, as we can see in the code snippet of the `mint` function below, highlighted in a red box.

![The line of code in mint where the callback happens](https://r2media.rareskills.io/UniswapV3Positions/image2.png)

The address that calls the `mint` function must implement the `uniswapV3MintCallback` function, as this is when the caller must transfer the token amounts required to modify the position. The `mint` function checks the pool's balance immediately before and after calling `uniswapV3MintCallback`, and accounts for the difference.

That is why the `mint` function cannot be called by an EOA: EOAs cannot respond to the callback to transfer the required token amounts. Thus, an EOA call to the `mint` function will revert if the LP is required to transfer any tokens, which is always the case (it is not allowed to modify a position by adding zero liquidity).

The transaction flow is illustrated below, assuming the contract that calls the `mint` function is named Position Manager.

![A visual representation of the Position manager and Uniswap V3 Core](https://r2media.rareskills.io/UniswapV3Positions/image1.png)

The core contract is NOT user-friendly. In general, the LP wants to choose the amount of tokens X and/or tokens Y they wish to deposit to open a position, rather than specifying the liquidity. The role of the intermediary contract is to provide a user-friendly interface, where the user chooses a token amount and the Position Manager converts this into the corresponding liquidity for that amount.

A representation of this process is shown below. The end user (an EOA) calls the `mint` function in the intermediary contract (Position Manager), passing the amount of tokens they want to transform into liquidity as one of the parameters. The position manager then converts this amount of tokens into liquidity and opens a position for the end user by calling the `mint` function in the core contract.

![A diagram showing how users interact with the position manager](https://r2media.rareskills.io/UniswapV3Positions/image5.png)

We will not go into the details of how a Position Manager works in this book. Uniswap provides a contract that works as a Position Manager, named [NonfungiblePositionManager](https://github.com/Uniswap/v3-periphery/blob/main/contracts/NonfungiblePositionManager.sol), in its [periphery library](https://github.com/Uniswap/v3-periphery), located in a different repository than the [core library](https://github.com/Uniswap/v3-core). 

## Summary

- Adding liquidity in Uniswap v3 means opening or modifying a position. Positions are defined by their owner’s address and their lower and upper ticks.
- Positions are stored in a mapping called `positions`, whose key is the Keccak hash of the owner address and the lower and upper ticks, and whose value is a struct located in the [`Position`](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/Position.sol) library.
- To open or add liquidity to a position, the `mint` function is used. This function is not user-friendly and receives, as an argument, the amount of liquidity the user wants to deposit between the lower and upper ticks.
- The `mint` function must be called by other contracts, not by EOAs. These intermediary contracts act between EOAs and the core contract, and one of their functions may be to convert a token amount defined by the LP into a liquidity amount expected by the core contract’s `mint` function.
- To deposit liquidity $L$ between a lower and an upper tick, it is necessary to deposit the amount of tokens corresponding to the real reserves of a segment with liquidity $L$ between these ticks.
