# Square Root Price in Uniswap V3

In Uniswap V2, the protocol tracks token reserves and derives the spot price, $p_x=y/x$, and total liquidity, $L=xy$, where $x$ and $y$ are the reserves of tokens X and Y.

Uniswap V3, instead, tracks the current price and liquidity, and derives the reserves. This calculation is complex and will be covered in later chapters.  The goal of this chapter is to explain how the protocol stores the token price.

Uniswap V3 actually stores the square root of the price, $\sqrt{p}$, instead of the price itself. This approach improves gas efficiency, as we will examine in detail in a later chapter. We don’t lose accuracy doing this, as the price can always be derived from its square root.

The goal of this chapter is to discuss where and how Uniswap V3 stores the square root of the price, $\sqrt{p}$, and how it handles the problem that the token price can be a decimal value, while Solidity has no float or decimal number type.

## The variable sqrtPriceX96

The square root of the price is stored in the field variable `sqrtPriceX96` of the struct `slot0` in the [pool’s contract](https://github.com/Uniswap/v3-core/blob/main/contracts/UniswapV3Pool.sol#L56-L74), as shown in the image below. 

![slot0 code screenshot](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/SqrtPriceX96-images/slot0.png)

The struct `slot0` also stores other variables, such as `tick`, which represents the current tick, as we saw in the [chapter on ticks](https://www.rareskills.io/post/uniswap-v3-ticks). The other variables in `slot0` relate to the oracle, fees, or contract security and will be examined later.

The variable name `sqrtPriceX96` already indicates that the stored value is the square root of the price (`sqrtPrice`) in a Q96 number format (`X96`), specifically the Q64.96 format.

A detailed explanation of the [Q number format](https://www.rareskills.io/post/q-number-format) was provided in the previous chapter. We will briefly review its usage in Uniswap V3 in the following section.

## The square root of the price, $\sqrt{p}$, as a fixed-point value

Uniswap V3 stores the square root of the price as a fixed-point number.

By way of review, fixed-point numbers allow us to gas-efficiently represent fractional numbers. For example, suppose we need to store `1.0050122696230506` but can only store integers. One approach is to multiply the value by a large number, such as $2^{96}$, resulting in:

$$
1.0050122696230506 \times 2^{96}=79625275426524700982079509374.66678
$$

We then discard the decimal portion and store `79625275426524700982079509374`.  

Thus, the relationship between the fixed-point representation Q64.96 of a number and the (original) number is

$$
\text{value\_in\_Q64.96} = \text{floor}(\text{original\_value} \times 2^{96}) 
$$

To recover the original value, we simply divide the fixed-point representation by $2^{96}$, as

$$
\text{original\_value} \approx \frac{\text{value\_in\_Q64.96}}{2^{96} } 
$$

In our example, the value `1.0050122696230506` is represented as a fixed-point number: `79625275426524700982079509374`. To recover the original value, we divide this value by $2^{96}$,  yielding approximately `1.0050122696230507`, which is very close to the original number.

This is exactly what Uniswap V3 does to get around the fact that Solidity doesn't have a float or decimal type. The variable that holds the square root of the token price, `sqrtPriceX96` , is a fixed-point number obtained by multiplying the actual, possibly fractional square root of the price $\sqrt{p}$, by $2^{96}$.

## The relationship between $\sqrt{p}$ and `sqrtPriceX96`

The relationship between $\sqrt{p}$ and `sqrtPriceX96` is that $\sqrt{p}$ is the actual square root of the price, while `sqrtPriceX96` represents the square root of a token's price in the Q64.96 format.

This relationship can be expressed with a simple formula. To convert $\sqrt{p}$ to `sqrtPriceX96`, we use:

$$
\text{sqrtPriceX96} = \text{floor}(\sqrt{p} \times 2^{96})
$$

To convert back `sqrtPriceX96` to $\sqrt{p}$, we use

$$
\sqrt{p} = \frac{\text{sqrtPriceX96}}{2^{96}}
$$

### Calculating `sqrtPriceX96` using Python

A Python code to convert a price to `sqrtPriceX96` and then retrieve the original price is below:

```python
from decimal import Decimal, getcontext
getcontext().prec = 100
price = Decimal('1.0050122696230506')
sqrtPriceX96 = price * Decimal(2) ** 96 # Decimal('79625275426524700982079509374.6667867672150016')
original_price = sqrtPriceX96 / 2 ** 96 # Decimal('1.0050122696230506')
```

The decimal library was used to increase calculation precision.

## Examples of values of `sqrtPriceX96`

Let's work through some examples.

1. If the current square root of the price of a token  $(\sqrt{p})$ is 100, this value will be stored in the variable `sqrtPriceX96`  as

$$
100 \times 2^{96}= 7922816251426433759354395033600
$$

1. If the current square root of the price is 323.002, it will be stored in `sqrtPriceX96` as:

$$
323.002 \times 2^{96} =  25590854948432409571389883046428
$$

To recover the original (and real) values, simply divide the above values by $2^{96}$.

## Querying sqrtPriceX96 in slot0 to find the price of a pool

### The ETH:DAI pool in Base

The current `sqrtPriceX96` value for the [ETH:DAI pool in Base](https://basescan.org/address/0x93e8542E6CA0eFFfb9D57a270b76712b968A38f5#readContract) can be seen in the image below as `4552234755200983230583166215033`.

![sqrtPrice screenshot](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/SqrtPriceX96-images/readSlot0.png)

To convert from `sqrtPriceX96` to $\sqrt{p}$, we divide the obtained value by $2^{96}$. So:

$$
\sqrt{p} =\frac{4552234755200983230583166215033}{2^{ 96}}≈57.46942221802943
$$

From the square root of the price, we obtain the actual price by squaring the value:

$$
p = (\sqrt{p})^2 = 57.46942221802943^2=3302.7344900741346  
$$

This represents the value of Ether in terms of DAI at the time this part of the text was written.

### The USDC:ETH pool in mainnet

As another example, let's retrieve the `sqrtPriceX96` in the [USDC:ETH pool on mainnet](https://etherscan.io/address/0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640#readContract#F11). This example is different from the previous one for two reasons: first, the price is for USDC in terms of Ether, rather than Ether in terms of a stablecoin. Second, Ether has 18 decimal places, while USDC has only 6, unlike the previous example where both tokens had 18 decimal places.

As can be seen in the image, the value is `1506673274302120988651364689808458`.

![USDCETH pool](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/SqrtPriceX96-images/USDCETH.png)

The value for $\sqrt{p}$ can be calculated as 

$$
\sqrt{p}=\frac{1506673274302120988651364689808458}{2^{96}} = 19016.89028861243
$$

The price can be calculated as

$$
p = (\sqrt{p})^2= 19016.89028861243 ^2 \approx 361642116.2491218
$$

Since this is a USDC:ETH pool, what we have is the price of USDC in terms of ETH. The price of ETH in terms of USDC is given by the inverse of the obtained price:

$$
p_{y} = \frac{1}{p_x} \approx \frac{1}{361642116.2491218}  \approx 2.7651646616046713 \times 10^{-9}
$$

Lastly, USDC has 6 decimal places, while ETH has 18. We need to take this difference into account; to calculate the price of ETH in terms of USDC, we must multiply the above value by $10^{12}$.

$$
p \approx 2.7651646616046713 \times 10^{-9} \times 10^{12} \approx 2765.1646616046713
$$

This represents the value of Ether in terms of USDC at the time this part of the text was written. ETH has been extremely volatile at the time we are writing this.

## The maximum price in the protocol

Since the square root of the price is stored in a `Q64.96` format number, the largest whole number it can store is approximately $2^{64}$. The price corresponding to this square root can be obtained by squaring this largest whole number.

Thus, the largest square root of a price the protocol can work with is approximately $2^{64}$, and the corresponding largest price is slightly below $2^{128}$.

## The minimum price in the protocol

A fixed-point number derived from $2^{96}$ can represent fractions as small as $2^{-96}$. This is because $2^{-96}$, when multiplied by $2^{96}$, results in `1`, which can be stored as an integer.

Values smaller than $2^{-96}$ are out of range. For instance, $2^{-97}$, when converted to fixed-point by multiplying by $2^{96}$, becomes $2^{-97} \times 2^{96}= 2^{-1}$, or 0.5. Since only the integer part is retained, it is rounded down to `0`, causing the information of the original value to be lost.

So, in theory, the lowest price the protocol could work with is $(2^{-96})^2$ or $2^{-192}$. However, as we will see in the next chapter, it does not allow for such a low price.

The protocol imposes a symmetry between the largest square root of the price that a token can assume, which is $2^{64}$, and the smallest square root of the price that a token can assume, which is imposed to be $2^{-64}$. This symmetry makes sense, since the value of token X relative to Y is the inverse of the value of token Y relative to X.

## Why use Q64.96 to store the square root of the price

This is not an easy question to answer, and the protocol team could have chosen a different Q number format. Since they decided to pack the square root of the price together with the tick and other information in a single 256-bit storage slot, the space left for the square root of the price was 160 bits.

In the next section, we will show how using 64 bits for the whole number portion is enough to accommodate prices in a real-world scenario, allowing the protocol to support a pool where one coin is worth trillions of dollars while the other is worth only fractions of a cent.

## Price limits of a token in a real world scenario

As we have seen, the highest price the protocol can handle is approximately $2^{128}$ ($(2^{64})^2$), or roughly $10^{38}$ in order of magnitude. Since token prices are always relative to another token, the price difference between two tokens in a pool can span up to $10^{38}$ orders of magnitude.

We must also be careful to factor in the decimals when computing the price differences. For example, a token with 18 decimal places is stored in its contract as $10^{18}$ units, while a token with 8 decimal places is stored as $10^8$ units. Therefore, if these two tokens have the same value in dollars, their price difference in the pool will already have 10 orders of magnitude only due to the decimal places. 

Let's consider a real world example, a WBTC:PEPE pool. Currently, 1 WBTC is worth approximately 100,000 ($10^5$) dollars, while 1 PEPE is worth approximately 0.00001 $(10^{-5})$ dollars, which is a difference of 10 orders of magnitude. 

However, we must also consider the difference in decimal places. WBTC has 8 decimal places, while PEPE has 18 decimal places, resulting in a 10-order-of-magnitude difference, in addition to the 10-order-of-magnitude difference due to the price difference.

Thus, the smallest unit of the WBTC token (one Satoshi, or $10^{-8}$ WBTC) is worth $10^{20}$ (or one hundred quintillion) times more than the smallest unit of PEPE (which has no name but represents $10^{-18}$ PEPE). 

Twenty orders of magnitude is a significant difference, but the Uniswap V3 pool allows for up to 38 orders of magnitude of difference. Therefore, the price of WBTC relative to PEPE could increase by another 18 orders of magnitude before reaching Uniswap V3's limit.

Assuming PEPE remains at its current price (0.00001 dollars), the price of Bitcoin must reach one hundred sextillion  $(10^{23})$ dollars before the Uniswap V3 price limits are reached.  Similarly, assuming Bitcoin reaches 1 trillion dollars, PEPE could be as low as 0.0000000000000001 dollars (one-tenth of one-quadrillionth of a dollar) before the protocol's price limits are reached.  

Deriving these limits is an exercise for the reader.

## Summary

- Uniswap V3 stores the square root of a price in a Q64.96 number format. It does this because there are no floating-point numbers in Solidity, and working with fixed-point numbers in binary is gas-efficient.
- The highest price of a token that the protocol can store is approximately $2^{128}$. To maintain symmetry, the lowest price it can store is $2^{-128}$. Token prices in the pool cannot exceed these values.