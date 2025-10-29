# Fixed Point Arithmetic in Solidity (Using Solady, Solmate, and ABDK as Examples)

A fixed-point number is an integer that stores only the numerator of a fraction — while the denominator is implied.

This type of arithmetic is not necessary in most programming languages because they have floating point numbers. It is necessary in Solidity because Solidity only has integers, and we often need to perform operations with fractional numbers.

Fixed point numbers are found in most DeFi smart contracts, so understanding them is a must.

For example, if the "implied denominator" is 100, then a fixed-point number holding "10" is interpreted as 0.1.

The most common fixed point number in Solidity is $10^{18}$: the number of "decimals" Ethereum and most ERC-20 tokens have. When we read the balance of an Ethereum address, we implicitly divide the number by $10^{18}$ to determine their Ether amount. For example, an address that has a balance of $10^{19}$ is interpreted to have 10 Ether — because the division by $10^{18}$ is implied.

The fixed point number with a denominator of $10^{18}$ is so common that engineers in the Solidity community refer to it as a "Wad" (the name was first introduced by MakerDAO). Sometimes, an 18-digit fixed-point number is interpreted as having the rightmost 18 digits allocated for the decimals, for example, the number "10" is shown below:

$$10 \; \underbrace{000000000000000000}_{\displaystyle 18 \text{ zeros}}$$

However, we have found this mental model makes understanding fixed-point arithmetic harder to learn, so this article will use the mental model of the fixed-point number holding the numerator, and the $10^{18}$ denominator being implied.

In this article we will learn how to do arithmetic with fixed point numbers and explain how popular fixed point libraries work.

## Converting an integer to a fixed point number

To convert an integer to a fixed point number, multiply the integer by the implied denominator. For example, "2 ether" is $2 \times 10^{18}$, so converting the integer 2 to "2 ether" we multiply it by $10^{18}$. The implied denominator of $10^{18}$ cancels out the $10^{18}$.

## Multiplying Fixed Point Numbers

To multiply two fixed-point numbers together, we follow the rules of multiplying fractions:

1. multiply the numerators together
2. multiply the denominators together
3. simplify the result.

For example:

$$
\frac {3} {4} \times \frac {2} {3} = \frac {6} {12} = \frac {1} {2}
$$

However, we can optimize this calculation in practice because the denominator is always the same with fixed point numbers.

Now let's consider a different set of fractions that have a common denominator:

$$
\frac{x}{d} \times \frac{y}{d} = \frac{z}{d^2}
$$

However, we do not want to return a result with an implied denominator of $d$​ because that would not be compatible with the implied denominator we selected. Therefore, we need to divide the numerator and denominator by $d$ to return a fixed point number consistent with our choice of denominator.

$$
\frac{x}{d} \times \frac{y}{d} = \frac{z}{d^2} \div \frac{d}{d} = \frac{z/d}{d}
$$

Thus, if $x$ and $y$ are fixed point numbers with an implied denominator $d^2$ we can compute their product as $(x \times y)/d$.

### Code example of multiplying fixed points

The Solady library has a `mulWad` math operation to multiply two fixed point numbers together with an implied Wad denominator ($10^{18}$). Below, we show the code and then explain how it relates to our previous discussion:

![solady mulwad function code](https://static.wixstatic.com/media/706568_22d5fb719d1b4b309663f39aecb1f0af~mv2.png/v1/fill/w_740,h_490,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_22d5fb719d1b4b309663f39aecb1f0af~mv2.png)

The core algorithm is at the bottom of the screenshot (inside the green box). We compute $(x \times y)/d$ there where $d$ is `WAD` or $10^{18}$ (as shown at the top of the screenshot where `WAD` is declared).

### Real world example

Suppose a user has 1 DAI (which has 18 decimals) and we wish to compute their balance assuming their deposit has earned 15% interest. This is a clear example of the need for fixed-point arithmetic, since we cannot directly multiply a number by 1.15 in Solidity.

![example contract using the solady mulwad library](https://static.wixstatic.com/media/706568_af9c7cafc7bc4d72a1f71e776a0d5b27~mv2.png/v1/fill/w_740,h_187,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_af9c7cafc7bc4d72a1f71e776a0d5b27~mv2.png)

The output is 1.15 after dividing by 1e18. Of course, we cannot actually divide by 1e18 because that would erase the decimals. We need a fixed-point representation because 1.15 cannot be represented as an integer. The code above can be [tested on Remix here](https://remix.ethereum.org/#code=aW1wb3J0ICJodHRwczovL2dpdGh1Yi5jb20vVmVjdG9yaXplZC9zb2xhZHkvYmxvYi9tYWluL3NyYy91dGlscy9GaXhlZFBvaW50TWF0aExpYi5zb2wiOwoKY29udHJhY3QgQyB7CgogICAgdXNpbmcgRml4ZWRQb2ludE1hdGhMaWIgZm9yIHVpbnQyNTY7CgogICAgdWludDI1NiB0b2tlbkJhbGFuY2UgPSAxZTE4OwoKICAgIGZ1bmN0aW9uIGNvbXB1dGUxNVBJbnRlcmVzdCgpIHB1YmxpYyB2aWV3IHJldHVybnMgKHVpbnQyNTYpIHsKICAgICAgICByZXR1cm4gdG9rZW5CYWxhbmNlLm11bFdhZCgxLjE1ZTE4KTsKICAgIH0KfQ&lang=en&optimize=false&runs=200&evmVersion=null&version=soljson-v0.8.25+commit.b61c2a91.js).

## Multiplying a Fixed-Point Number by an Integer

Multiplying a fraction $x$ by an integer $y$ is the same as multiplying $x$ by $y/1$:

$$
\frac{35}{100} \times 3 = \frac{35}{100} \times \frac{3}{1} = \frac{105}{100}
$$

Thus, when we multiply a fixed-point number by an integer, we do not need any extra steps. We simply interpret the return value as a fixed-point number with the denominator unchanged.

## Dividing Fixed Point Numbers

To divide fractions, we "flip" the second fraction and multiply them together. For example:

$$
\begin{split}   &\frac{2}{5} \div \frac{1}{2}   \\
                                                 \\
                =  &\frac{2}{5} \times \frac{2}{1}\\
                                                   \\
                = &\frac{4}{5}
\end{split}
$$

Now let's consider an example where they have the same denominator:

$$
\frac{6}{10} \div \frac{3}{10} = \frac{6}{10} \times \frac{10}{3} = \frac{60}{30} = 2
$$

Note that the common denominator of 10 got cancelled. If we want to represent 2 with an implied denominator of 10 (i.e. as a fixed point number with denominator 10), we need to multiply it by 10 again:

$$
2 = \frac {2 \times 10} {10}
$$

Thus, for a general $x$ and $y$ having a common denominator $d$, if we want to express the output with an implied denominator of $d$, we must do the following:

$$
\frac{x}{d} \div \frac{y}{d} = \frac{x}{d} \times \frac{d}{y} = \frac{x}{y} = \frac{(x \times d) /y}{d}
$$

Thus, if $x$ and $y$ are fixed point numbers with an implied denominator $d$ we can compute their quotient as $(x \times d)/y$.

```solidity
/// @dev Equivalent to `(x * WAD) / y` rounded down.
function divWad(uint256 x, uint256 y) internal pure returns (uint256 z) {
    /// @solidity memory-safe-assembly
    assembly {
        // Equivalent to `require((y != 0 && (WAD == 0 || x <= type(uint256).max / WAD))`.
        if iszero(mul(y, iszero(mul(WAD, gt(x, div(not(0), WAD)))))) {
            mstore(0x00, 0x7c5f487d) // `DivWadFailed()`.
            revert(0x1c, 0x04)
        }
        z := div(mul(x, WAD), y)
    }
}
```

If we put `mulWad()` and `divWad()` side-by-side, we can see the only difference between them (at the computation step, not the overflow check), is that the div case is multiplying by an inverted fraction.

![solady mulwad() and muldiv() code side by side](https://static.wixstatic.com/media/706568_ebdad9cd9eac42bb8b7174008b20cbad~mv2.jpeg/v1/fill/w_740,h_148,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/706568_ebdad9cd9eac42bb8b7174008b20cbad~mv2.jpeg)

## Dividing a Fixed Point Number by an Integer

Suppose we want to divide 2.5 by 2 (or some fraction by an integer in general). It is not necessary to convert 2 to a fixed-point number via $(2 \times d)/d$.

Dividing a fraction $x$ by an integer $y$ is the same as dividing the numerator of $x$ by $y$.

$$
\frac{35}{100} \div 3 = \frac{35 \div 3}{100} = \frac{11}{100}
$$

Note that $35 \div 3 = 11$, not 11.666, because we are using integer division, not floating points. We simply divide the fixed-point number by the integer and interpret the result as a fixed-point number. As with multiplying a fixed-point number by an integer, the denominator remains the same.

## Adding and Subtracting Fixed Point Numbers

Adding and subtracting fractions that have the same denominator are simply added together and the denominator is ignored. We interpret the sum as a fixed-point number with the same implied denominator as the summands. For example,

$$
\frac {a}{d} + \frac {b}{d} = \frac {a+b}{d}
$$

Therefore, when adding fixed point numbers of the same denominator, we simply add the numbers together like we would with regular integers.

Consider the example with an implied denominator of 100:

$$
\frac {50}{100} - \frac {40}{100} = \frac {10}{100}
$$

To compute it, we simply do $50 - 40 = 10$, we don't need to incorporate 100 into our computation.

## Binary vs Decimal Fixed Point Number

A binary fixed point number is a fixed point number where the denominator can be expressed as $2^n$. Binary fixed point numbers are usually denoted with Q notation. For example, UQ112x112 uses $2^{112}$ as the denominator. The U means "unsigned." The datatype used to hold UQ112x112 would be 224. Another way to interpret this is that the "fractional part after the decimal" is held in the rightmost 112 bits and the "whole number part" is held in the left 112 bits.

As another example, UQ64x64 (or UQ64.64) is a `uint128` that holds the "fractional part" in the least significant 64 bits and the "whole number" in the most significant bits. This can still be interpreted as having an implied denominator of $2^{64}$ as we will see next.

The advantage of a binary fixed point number is we can use a gas-efficient left bit shift instead of multiplying by the denominator, (when converting an integer to a fixed point number, or doing a right bit shift when dividing.

As a basic example, consider that:
(1) 2 has a binary representation of 10
(2) 16 has a binary representation of 10000
(3) $16 = 2 \times 2^3$
(4) binary(1000) = binary(10) << 3

Note that 3 is the exponent in (3) and the amount we left shift the bits by in (4).

The relationship between the bitshift by amount e and multiplying by $2^e$ holds in general. The following operations are equivalent:

```solidity
// x * 2¹¹² equals x left bitshifted by 112 bits
x * 2 ** 112 == x << 112

// x / 2¹¹² equals x right bitshifted by 112 bits
x / 2 ** 112 == x >> 112
```

`x` can be an arbitrary number as long as it fits in the unsigned integer.

The [ABDK library](https://github.com/abdk-consulting/abdk-libraries-solidity/tree/master) converts unsigned integers to fixed point numbers (with an implied $2^{64}$ denominator) using the following function:

![ABDK library fromUInt function code](https://static.wixstatic.com/media/706568_3afa606cd5934a618753eaa9bde76fd5~mv2.png/v1/fill/w_740,h_338,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_3afa606cd5934a618753eaa9bde76fd5~mv2.png)

The require statement ensures that `x` is less than `type(int64).max`, since the ABDK library uses signed fixed point numbers. The left shift by 64 is equivalent to multiplying by $2^{64}$.

Similarly, when ABDK does a multiplication, instead of dividing the product of `x` and `y` by $2^{64}$, it does a right shift by 64 bits:

![ABDK mul function code](https://static.wixstatic.com/media/706568_0696d658a648469086e1b07256603777~mv2.png/v1/fill/w_740,h_410,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_0696d658a648469086e1b07256603777~mv2.png)

### Uniswap V2 Fixed Point Library

The fixed point library of [Uniswap V2](https://rareskills.io/uniswap-v2-book) is quite simple because the only operations Uniswap V2 does with fixed point numbers is addition and division of a fixed point number by an integer.

```solidity
pragma solidity =0.5.16;

// A library for handling binary fixed point numbers (https://en.wikipedia.org/wiki/Q_(number_format))

// range: [0, 2**112 - 1]
// resolution: 1 / 2**112

library UQ112x112 {
    uint224 constant Q112 = 2**112;

    // encode a uint112 as a UQ112x112
    function encode(uint112 y) internal pure returns (uint224 z) {
        z = uint224(y) * Q112; // never overflows
    }

    // divide a UQ112x112 by a uint112, returning a UQ112x112
    function uqdiv(uint224 x, uint112 y) internal pure returns (uint224 z) {
        z = x / uint224(y);
    }
}
```

The `encode()` function converts a `uint112` to a fixed point number stored in a `uint224`. Uniswap V2 uses an implied denominator of `2**112`. It could be more gas efficient if it used a bitshift instead of a multiplication (this is probably a mistake by the Uniswap developers).

The fixed point number is stored in a `uint224` which is twice the size of the `uint112` it will interact with. During the encode operation, the bits of the `uint112` number are effectively shifted to the most significant 112 bits of the `uint224`.

This "encode" operation is easier to visualize with smaller uint sizes. Let's use a hypothetical fixed-point number with a $2^8$ denominator. Below, we show what happens when we encode a `uint8` to a fixed point number with a $2^8$ denominator:

<video src="https://video.wixstatic.com/video/706568_91fa05a2c70a479a9846b50f7cb946de/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

Starting with the number 125 which has binary representation `01111101`, if we multiply it by $2^8$, the product is 32000, which when stored in a 16 bit uint is represented as `0111110100000000`. Note that multiplying 125 by $2^8$ has the same effect as shifting left by 8 bits.

The `uqdiv()` function simply does division of a fixed point number by an integer, which needs no extra steps.

Uniswap uses this library to accumulate the prices for the TWAP oracle below. The TWAP adds the latest price every time an update happens into an accumulator (which is used to calculate the average price with additional steps, something outside the scope of this article). Since prices are represented as fractions, fixed point numbers are an ideal way to represent them.

The variables `_reserve0` and `_reserve1` hold the most recent token balances of the pool and are uint112. `price0CumulativeLast` and `price1CumulativeLast` are UQ112x112 (fixed point numbers with an implied denominator of $2^{112}$). The code below from Uniswap V2 converts the numerator to a fixed point number (UQ112x112) and divides it by an integer (the denominator is not converted to a UQ112x112). The result is a fixed point number.

![_update() function in uniswap using the UQ112X112 encode function](https://static.wixstatic.com/media/706568_65bc95ef47c64c9e8f3e88cfbb922b37~mv2.png/v1/fill/w_740,h_301,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_65bc95ef47c64c9e8f3e88cfbb922b37~mv2.png)

## Rounding up vs rounding down

Fixed-point libraries commonly have the option to round up when dividing. For example, Solady has:

- `mulWadUp` — multiply two fixed point numbers, but when dividing by d, round up. Recall, the formula for multiplying two fixed point numbers is $(x \times y) / d$.
- `mulDivUp` — divide two fixed point numbers, but round up when dividing

Solidity division always rounds down, for example $10 / 3 = 3$. However, if we were to round up, $10 / 3$ would be equal to 4. When calculating credits or prices, one should always round in favor of the protocol and against the user. For example, if we are computing how much a user should pay for a fixed amount of another asset, we should round the price up.

For example:

- $10 / 3$ rounding down is 3.3333
- $10 / 3$ rounding up is 3.3334 (depending on the size of our denominator)

Rounding up simply means adding 1 to the result if the remainder is non-zero. For example, $9 / 3 = 3$ exactly, so we should not return 4. However, $10 / 3$ and $11 / 3$ have a remainder of 1 and 2 respectively, so we should add 1 to the result of the division.

Here is how the [Solmate library](https://github.com/transmissions11/solmate/blob/main/src/utils/FixedPointMathLib.sol) does it:

![Solmate mulDivUp function](https://static.wixstatic.com/media/706568_92f362641ea849818e7cc8672e38f537~mv2.png/v1/fill/w_740,h_389,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_92f362641ea849818e7cc8672e38f537~mv2.png)

In the green underline, the code checks if the modulo is greater than zero. If it is, then 1 is added to the result (rounding up), otherwise 0 is added (no round up).

*Originally Published Jun 10*
