# Computing the Current Tick Given sqrtPriceX96

In the previous chapters, we saw that the protocol stores the square root of the price instead of the price itself. Therefore, it is necessary to relate ticks to the square root of the price in the fixed-point Q64.96 format.

In this chapter, we will explore how to convert between `sqrtPriceX96` and ticks.

## The protocol stores tick indexes

A tick is a discrete price, given by the formula $p(i)=1.0001^i$, where $i$ is called the tick index or simply tick. Having the tick index, it is possible to uniquely determine the tick price, and vice versa. Since storing tick indexes $i$ occupies fewer bits than storing $p(i)$, the protocol stores tick indexes instead of tick prices.

In this book, we will use the terms *tick* (price) and *tick index* interchangeably. In the codebase, variables named tick always refer to tick index $i$.

## Compute tick index, Given `sqrtPriceX96`

We begin by recalling the relationship between the square root of the price and the variable `sqrtPriceX96`:

$$
\left( \frac{\text{sqrtPriceX96}}{2^{96}}\right) = \sqrt{p} 
$$

Thus, the relationship between the price and `sqrtPriceX96` is given by taking the square of both sides.

$$
\left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)^2 = p 
$$

Since the relation between price and tick index is $p = 1.0001^i$, we have that

$$
\left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)^2 = 1.0001^i
$$

To determine the tick index $i$ in terms of `sqrtPriceX96`, we take the logarithm of both sides of the equation above.

$$
\begin{align*}
\log \left( \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)^2 \right) &= \log (1.0001^i)  \\  2 \, \log \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right) &= i \, \log(1.0001) && \text{used that}  \;\log(a^b) = b \; \log(a)    \\  i &= \frac{2 \, \log \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)}{\log(1.0001)} && \text{divided both sides by } \log (1.0001)
\end{align*}
$$

A logarithm is always taken with respect to a base. For example, if $b^a=c$, then $\log_b{c}=a$ (log is taken in base b). However, the logarithm in the equation above can be taken in any base. The reason for this is explained in the last section of this chapter.

### The tick index is a discrete value

Actually, the above formula is not entirely correct for one reason: `sqrtPriceX96` is a continuous value, while the tick index `i` is discrete (an integer). Therefore, the value needs to be rounded down. 

This can be seen in the illustration below, where we use a tool that can be found in our article on ticks. Note that while the price changes continuously, the corresponding tick for that price is always the one immediately below it.

<video src="https://r2media.rareskills.io/UniswapV3TickToPrice/lower-tick-anim.mp4" type="video/mp4" autoplay loop muted controls></video>

Thus, the accurate formula for tick is:

$$
i = \lfloor \frac{2 \, \log \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)}{\log(1.0001)} \rfloor
$$

where the symbol $\lfloor ...\rfloor$ means to round down, for instance, $\lfloor 3.14 \rfloor = 3$.

## Compute `sqrtPriceX96` given the tick index

To calculate the square root of the price from the tick index, we start with the formula

$$
\left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)^2 = 1.0001^i
$$

Then, take the square root of both sides and multiply by $2^{96}$,

$$
\begin{align*} 
\frac{\text{sqrtPriceX96}}{2^{96}} &= \sqrt{1.0001^{i}} \\ \text{sqrtPriceX96} &= \sqrt{1.0001^{i}} \cdot 2^{96}
\end{align*}
$$

## From ticks to price and vice-versa in the codebase

The conversion between ticks and `sqrtPriceX96` in Solidity is handled by the [TickMath library](https://github.com/Uniswap/v3-core/blob/main/contracts/libraries/TickMath.sol), which will be studied in a separate chapter due to the complexity of implementing log and square root in Solidity.

It contains two functions: `getSqrtRatioAtTick` and `getTickAtSqrtRatio`, which perform the conversion between `sqrtPriceX96` and its corresponding tick index, as well as the opposite. This can be seen below.

![Functions for converting between tick and price](https://r2media.rareskills.io/UniswapV3TickToPrice/getSqrtRatioAtTick.png)

We saw the mathematics of these formulas in the previous section. In this section, we will perform this calculation using Python. 

### The `getSqrtRatioAtTick` function

The `getSqrtRatioAtTick` function in Python can be written as:

```python
def getSqrtRatioAtTick(i):
    return math.sqrt(1.0001 ** i) * 2**96    
```

As an example, the value of `sqrtPriceX96` for the upper bound price can be calculated using `getSqrtRatioAtTick(887272)`, resulting in `1.4614467034780703e+48`, which (approximately) corresponds to the `MAX_SQRT_RATIO` constant from the `TickMath` library.

![Max ratio constant in TickMath](https://r2media.rareskills.io/UniswapV3TickToPrice/tickMathMaxRatio.png)

### Using the Decimal library

To get a more accurate result, we can use the `Decimal` library in Python. The same formula, using this library, is coded below, together with the calculated value for `MAX_SQRT_RATIO`.

```python
import math
from decimal import Decimal, getcontext

getcontext().prec = 50 

def getSqrtRatioAtTick(i):
    base = Decimal("1.0001")
    exponent = Decimal(i)
    sqrt_value = base ** (exponent / 2) 
    multiplier = Decimal(2) ** 96
    return sqrt_value * multiplier
    
print(int(getSqrtRatioAtTick(887272)))  # 1461446703485210103244672773810124308346321380902
```

### The `getTickAtSqrtRatio` function

The `getTickAtSqrtRatio` function in Python can be written as:

```python
def getTickAtSqrtRatio(sqrtPriceX96):
    return math.floor(2*math.log(sqrtPriceX96/2**96)/math.log(1.0001))
```

As an example, the tick corresponding to the lower bound of `sqrtPriceX96`, `4295128739`, can be calculated using `getTickAtSqrtRatio(4295128739)`, resulting in the value `-887272`, which is expected for the lower bound for ticks.

```jsx
getTickAtSqrtRatio(4295128739) // -887272
```

## Changing the base of the logarithm

When we derived the formula

$$
i = \lfloor \frac{2 \, \log \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)}{\log(1.0001)} \rfloor
$$

we mentioned that the logarithm can be in any base. In the implementation of the formula in Python, we used the natural logarithm, but we could also use the logarithm in base 10 or any other, which would yield the same result.

The reason is a basic property of logarithms that relates logarithms in different bases,

$$
\log_{\boxed{b}}(k) = \frac{\log_a(k)}{\log_a(\boxed{b})}
$$

where $a$ and $b$ are two different bases, and $k$ is the argument of the logarithm we want to calculate. For instance, converting a log from natural base $e$ to base 10, we have

$$
\log_{10}(k) = \frac{\log_e(k)}{\log_e(10)}
$$

The catch is that the divisor, $\log_e{(10)}$, depends only on the bases and not on the argument. Suppose we have a fraction:

$$
\frac{\log_{10}(x)}{\log_{10}(y)}
$$

If we want to convert the logarithms to some arbitrary base b, we divide both the numerator and denominator by $\log_b(10)$. However, dividing both the numerator and denominator of a fraction by the same value has no effect on the final answer because it cancels out.

Let’s make this explicit by applying this relationship to our formula for tick index $i$ and converting from the natural base to base 10:

$$
\begin{align*}
i &= \lfloor \frac{2 \, \log_e \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)}{\log_e(1.0001)} \rfloor \\i  &= \lfloor \frac{2 \, \log_{10} \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right) / \log_{10}(e)}{\log_{10}(1.0001) / \log_{10}(e)} \rfloor && \text{from base e to base 10} \\ i  &= \lfloor \frac{2 \, \log_{10} \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right) / \cancel{\log_{10}(e)}}{\log_{10}(1.0001) / \cancel{\log_{10}(e)}} \rfloor  \\  i &= \lfloor \frac{2 \, \log_{10} \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)}{\log_{10}(1.0001)} \rfloor
\end{align*}
$$

As we can see, the transformation factor cancels out and the calculation is the same regardless of the base.

## Summary

- The protocol needs to convert between `sqrtPriceX96` and the tick index. This is done using the `getSqrtRatioAtTick` and `getTickAtSqrtRatio` functions, which are located in the `TickMath` library.
- To convert from `sqrtPriceX96` to the tick index, the following formula should be used:

$$
i = \lfloor \frac{2 \, \log \left( \frac{\text{sqrtPriceX96}}{2^{96}}\right)}{\log(1.0001)} \rfloor
$$

- To convert from tick index to sqrtPriceX96, the following formula should be used:

$$
\sqrt{1.0001^{i}} \cdot 2^{96}
$$

## Practice Exercises

Look through the [Uniswap V3 Pools](https://app.uniswap.org/explore/pools/ethereum) to find the pool address, then open the address in a block explorer. Look for the public variable `slot0` and convert the `tick` to `price` and vice versa.

![Etherscan output of slot0](https://r2media.rareskills.io/UniswapV3TickToPrice/etherscanSlot0.png)

Check that your calculation is close to what Uniswap provides and then convert the sqrtPriceX96 to dollars.

Do it for the following pools:

- USDC/ETH on
    - [Uniswap Pool UI](https://app.uniswap.org/explore/pools/ethereum/0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640)
    - [Etherscan contract link](https://etherscan.io/address/0x88e6A0c2dDD26FEEb64F039a2c41296FcB3f5640#readContract)
- ETH/USDC on Base
    - [Uniswap Pool UI](https://app.uniswap.org/explore/pools/base/0xd0b53D9277642d899DF5C87A3966A349A798F224)
    - [Etherscan contract link](https://basescan.org/address/0xd0b53D9277642d899DF5C87A3966A349A798F224#readContract)
- WBTC/USDC on mainnet
    - [Uniswap Pool UI](https://app.uniswap.org/explore/pools/ethereum/0x99ac8cA7087fA4A2A1FB6357269965A2014ABc35)
    - [Etherscan contract link](https://etherscan.io/address/0x99ac8cA7087fA4A2A1FB6357269965A2014ABc35#readContract)

We will learn later why it is not safe for a contract to directly consume `sqrtPriceX96` (which can give us the price) and `tick` data from `slot0`, this is only for practice at this point.
