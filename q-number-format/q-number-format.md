# Q Number Format

Q number format is a notation for describing binary [fixed-point numbers](https://www.rareskills.io/post/solidity-fixed-point).

A fixed-point number is a popular design pattern in Solidity for storing fractional values, since the language does not support floating point numbers. Thus, to “capture” the fractional portion of a number, we multiply the fraction by a whole number so that the fractional portions become whole numbers (with some potential precision loss).

## Ether decimals vs Q Numbers

The most well-known fixed point number in Solidity programming is “One Ether.”  One Ether is actually $10^{18}$ base units (Wei). “One” Ether is actually 1 times $10^{18}$. Since we cannot store “0.5” in Solidity, we instead store $\space0.5\times10^{18}=5\times10^{17}$ or 500000000000000000. Essentially, $10^{18}$ multiplies by 0.5 so that the fraction gets “preserved” as an integer. This is not a perfect solution, however, since it cannot hold values smaller than $10^{-18}$. However, $10^{18}$ is good enough for most applications, and contracts can use a larger number if more precision is needed.

A Q number, on the other hand, multiplies the fraction by a power of 2 instead of a power of 10 because multiplying (and dividing) by powers of 2 is more gas efficient in the EVM, since multiplying or dividing by powers of two can be done using bitshift operations. For example, `x << n` is equivalent to `x * 2**n` and `x >> n` is equivalent to `x / 2**n`.

## Q Number Notation

A Q number is often written in the form `Qm.n` where n is the power of 2 used to multiply the number by and `m` is the unsigned integer size of the integer portion. This is completely equivalent to saying `m` is the number of bits used to store the integer portion and `n` is the number of bits used to store the fraction.

To demonstrate the equivalence, consider a Q1.1 number. This number allocates 1 bit for the integer portion (m = 1) and 1 bit for the fraction (n = 1). The number 1 would be represented as $1\times2^1$ which is equivalent to `1 << 1`. The binary representation of $1\times2^1$ or `1 << 1` is `10`. Note that our number is 2 bits large ($n + m = 2$). We could deconstruct the `10₂` as

$$
\underbrace{1}_\text{integer portion}\underbrace{0}_\text{ fractional portion}
$$

Thus, the variable holding “1” as a Q1.1 would actually store the value 2, or binary `10₂`. However, we “interpret” or “treat” the value 2 as 1 if the variable used for storing a Q1.1.

Let’s see an example with a Q8.4 number. The number 1 is represented as $1 \times2^4$ or `1 << 4`. The binary representation of `1 << 4` is `10000₂`. Since we have 8 bits representing the whole number portion, we need to leftpad with zeros until our entire number is 12 bits large:

$$
\underbrace{00000001}_\text{whole number portion }\underbrace{0000}_\text{ fractional portion}
$$

There are a total of 12 bits storing the number, since 8 + 4 = 12. “Under the hood” we are actually storing the value $2^{4}=16$, but we interpret the variable value as 1. This is similar to how we might interpret a variable as holding “1 Ether” but “under the hood” the variable is actually storing $10^{18}$. When we say “under the hood” we mean the actual number that is in memory or storage.

As a third example, consider a Q4.8 number. The number 1 is represented as $1\times2^8$ or `1 << 8`. The binary representation of `1 << 8` is `10000000₂`. Since we have 4 bits representing the integer portion, we leftpad with three zeros until the entire number is 12 bits large:

$$
\underbrace{0001}_\text{whole number portion }\underbrace{00000000}_\text{ fractional portion}
$$

The Q4.8 and Q8.4 numbers both require 12 bits to store. However, the Q8.4 number has more bits allocated to the integer portion, and hence it can store a larger integer. On the other hand, the Q4.8 number has more bits allocated to the fractional portion, so it can be more precise in representing fractions.

### The value “1” as a fixed point number

In general for a Q number, “one” is the 1 multiplied by $2^m$ where $m$ is the number of bits to store the fraction. So 1 in Q64.96 is $1\times2^{96}$ or `1 << 96`. “One” in Q128.128 is $1\times 2^{128}$ or `1 << 128`. “One” in Q64.128 is also `1 << 128` because we only care about the number of fractional bits.

For *any* `Qm.n`, the representation of the integer 1 is always `1 << n`, and it just so happens that *both* `Q64.128` and `Q128.128` have `n = 128`.

Note that `n` means number of *binary* bits (base 2), not “number of decimals” as in a power of ten (base 10). Qx.18 is not the same as how we use $10^{18}$ to store 1 ether. Qx.18 means “one” is $1\times2^{18}$ not $1\times10^{18}$.

### Using Q numbers in Solidity

The number of bits in a Qm.n number must be m + n bits large. Therefore:

- A Q64.64 number must at least use a uint128 (128 = 64 + 64)
- A Q64.96 number must at least use a uint160 (160 = 64 + 96)
- A Q128.128 number must use a uint256

## Interpreting the fractional portion of a Q number

Let’s consider all possible values of a Q1.1 number:

|  | “under the hood” value | float value (”under the hood” ÷ 2^1) |
| --- | --- | --- |
| 0 0 | 0 | 0 |
| 0 1 | 1 | 0.5 |
| 1 0 | 2 | 1.0 |
| 1 1 | 3 | 1.5 |

We can see that the “fraction bit” being set to 1 conveys that the single bit in the fraction portion represents the value 0.5. In general, the fraction bits represent fractional powers of two. Let’s see an example of a Q1.2 number as an example:

|  | “under the hood” value | float value (”under the hood” ÷ 2^2 |
| --- | --- | --- |
| 0 00 | 0 | 0 |
| 0 01 | 1 | 0.25 (1/4) |
| 0 10 | 2 | 0.5 (2/4) |
| 0 11 | 3 | 0.75 (3/4) |
| 1 00 | 4 | 1.00 (4/4) |
| 1 01 | 5 | 1.25 (5/4) |
| 1 10 | 6 | 1.50 (6/4) |
| 1 11 | 7 | 1.75 (7/4) |

In general, the bits in a Q number are interpreted as follows. Note that the bits to the right of the decimal represent fractional powers of two:

$$
\begin{matrix}
\dots&1 & 1 & 1 & 1 & . & 1 & 1 & 1 & 1&\dots\\
\dots&8 & 4 & 2 & 1 & . & \frac{1}{2} & \frac{1}{4} & \frac{1}{8} & \frac{1}{16}&\dots
\end{matrix}
$$

Each bit in the fraction additively represents a fractional power of two. For example, `0.1₂` represents 0.5, `0.11₂` represents 0.75, and `0.001₂` represents 0.125 or one eight.

Q numbers can only encode fractions that can be represented as sums of 1 over a power of 2. If we try to represent a number like 1/3, there must necessarily be a rounding error.

Try plugging in various fractional values into the interactive tool below to see how they are converted to a fixed point representation:

<iframe
  src="https://www.rareskills.io/fixed-point-demo"
  width="700"
  height="470"
  frameborder="0">
</iframe>

## Converting an integer to a Q number

To convert the *integer* 1 to a Q64.96 fixed-point number, we compute `1 << 96`. This creates a binary number with 1 in the 96th place and zeros in the binary values 0 to 95 inclusive.

In general, we can convert an integer to a Qm.n number by leftshifting the integer by `n` bits.

The animation below illustrates this:

<video src="https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/QNumbers/IntegerToQ44c.mp4" type="video/mp4" autoplay loop muted controls></video>

## Converting a Q number to an integer

A Q number has a fraction portion, but an integer does not. So to convert a Q number to an integer, we simply bitshift the integer portion over so that the fractional part is removed.

Therefore, if we want to extract the integer portion of a fixed point number, we rightshift the number by `n` bits (remember: `n` is the number of fractional bits in Qm.n). This causes the fractional bits to disappear, leaving us only with the integer portion. In other words, we are converting a fixed-point Q number to an integer by chopping off all the fractional bits.

Consider the following animation of converting a Q4.4 number to an integer:

<video src="https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/QNumbers/Q44ToInteger.mp4" type="video/mp4" autoplay loop muted controls></video>

## Constructing a fixed point value

Suppose we want to encode the number “1.5” as a Q64.96. Solidity does not accept `1.5 * 2**96` or `1.5 << 96` as valid syntax.

Instead, 1.5 can be computed as

```solidity
1 * 2**96 + 2**96 / 2; // equivalent to 1 + 0.5
```

The value stored “under the hood” for 1.5 as a Q64.96 number would be `118842243771396506390315925504`. We can compute this in Python as:

```python
>>> int(1.5 * 2**96)
118842243771396506390315925504
```

In binary, we can see that 96 bits are used for the fraction:

```python
>>> bin(118842243771396506390315925504)
'0b1 100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000' # space added for clarity
```

## Converting a Q number to a floating point number (off-chain)

Using our previous example of 1.5, we can divide 118842243771396506390315925504 by $2^{96}$ and get

$$
\frac{118842243771396506390315925504}{2^{96}}=1.5
$$

When we compute the float value of a Q number off-chain, we divide by $2^{96}$, not bitshift by 96. This is because bitshifting “destroys” the rightmost 96 bits, so the information containing the fraction would be lost.

## The largest value a Q number can hold

The largest *integer* value a Q number can hold is $2^m-1$ where $m$ is the number of bits in the integer portion. The largest *total* *value* it can hold is the largest integer portion plus the largest representable fraction:

$$
\sum_{i=1}^{n}=\frac{1}{2}+\frac{1}{4}+\dots+\frac{1}{2^n}
$$

This is mathematically equivalent to $1-\frac{1}{2^n}$. Therefore, the largest *total* value a Qm.n number can store is $2^m-\frac{1}{2^n}$.

For example, the *integer* largest value a Q96.64 is $2^{96} - 1$ or 79228162514264337593543950335. The largest value a Q96.64 value can store, including the fraction part, is

$$
(2^{96}-1) + (1 - \frac{1}{2^{64}})
$$

or 

79228162514264337593543950335.9999999999999999999457898913757247782996273599565029144287109375

## Adding two Q numbers together

If we want to add, for example, a Q128.128 number to a Q64.64 number, we have two options. The first option is to left bitshift the Q64.64 number by 64 bits so that the decimal points align. This effectively converts a Q64.64 number to a Q64.128 number.

Alternatively, we have a second option. If we don’t need 128 bits of precision, or we specifically want a Q64.64 number as the sum, we can right bitshift the Q128.128 number by 64 bits, which converts it to a Q128.64 number. Of course, this results in some precision loss.

To add two Q numbers together, we only need them to have the same number of bits in the fractional portion, i.e. the decimal points must be aligned. However, the “destination” data type must have a large enough whole-number portion to handle the sum, or we might have an overflow.

If we want to subtract two Q numbers together, we follow the same logic outlined in this section.

## Multiplying or dividing

If we multiply $1\times1$ we expect to get $1$ as the answer. Under the hood however, the number $1$ is represented as $2^{96}$. Therefore, if we multiply the number 1 represented as a Q64.96 by itself, we would actually be carrying out the operation $2^{96}\times2^{96}=2^{192}$, but we actually want $2^{96}$ as the answer.

Therefore, when we multiply two Q numbers together, we must follow up by shifting the product right by `n` bits. In our example of 96 bits, this means that $2^{192}$ would be rightshifted by 96 bits to become $2^{96}$.

## Some Solidity Fixed Point Examples

### Example 1: Divide 5 by 2

Suppose we want to divide 5 by 2 and return the result as a Q64.64 number. Since the Q64.64 number cannot hold integers larger than 64 bits, we may as well represent integers 5 and 2 using `uint64`. Actually holding a Q64.64 requires 128 bits, so we’ll use a `uint128` to hold Q64.64.

In other words the integers are represented with `uint64` since that is the largest integer a Q64.64 can hold, but the Q64.64 number is represented with a `uint128` since it needs to hold 64 bits for the integer and 64 bits for the fraction.

```solidity
function divToQ64x64(uint64 x, uint64 y) public pure returns (uint128) {
    // convert x (a uint64 integer)
    // to a Q64.64 fixed-point number by left-shifting 64 bits.
    uint128 x64_64 = x << 64;
			
    // divide by y
    return x64_64 / y;
}

divToQ64x64(5, 2); // returns 46116860184273879040
```

The result `46116860184273879040` encodes as `2.5` because `46116860184273879040 / 2**64 = 2.5`.

### Example 2: Multiply 5 by 0.5

Let’s multiply 5 and 0.5 together when both are represented as Q64x64. The representations are as follows:

- 5 as a Q64.64 number is `5 << 64` or `5 * 2**64` which equals 92233720368547758080.
- 0.5 as a Q64.64 number is `1 << 64 / 2` or `2**64 / 2` which equals 9223372036854775808

When we multiply two fixed point numbers together, we need to ensure they don’t overflow before we divide by $2^{64}$ to normalize them again. This means we will store the product in a `uint256` then rightshift by 64 bits.

```solidity
function mulU64x64(uint128 x, uint128 y) public pure returns (uint128) {
    // note: Solidity performs multiplication using uint128 unless
    // explicitly upcasted. This could overflow and revert. 
    uint256 temp = uint256(x) * uint256(y);
    return uint128(temp >> 64);
}

mulU64x64(5 * 2**64, 2**64 / 2) // returns 46116860184273879040
```

This returns the expected result since 46116860184273879040 / 2**64 = 2.5.

### Example 3: Multiply 5 by 0.5, but 5 is an integer instead of a fixed point

As a variation to the example above, we want to multiply 5 (an integer) by 0.5 (a fixed point number) and return a fixed point number. The only difference from the code above is converting the 5 to its fixed point representation:

```solidity
function mulUint64ByQ64x64(uint64 x, uint128 y) public pure returns (uint128) {
        
    // convert uint64 to fixed point
    uint128 x_fp = uint128(x) << 64;
        
    uint256 temp = uint256(x_fp) * uint256(y);
    return uint128(temp >> 64);
}

mulUint64ByQ64x64(5, 2**64 / 2) // returns 46116860184273879040
```

The result is still the fixed point equivalent of 2.5.

It is a bit wasteful to left-shift by 64 and then right-shift later. The same computation above can be done more efficiently as:

```solidity
function mulUint64ByQ64x64(uint64 x, uint128 y) public pure returns (uint128) {

    return uint128(uint256(x) * uint256(y));
}

mulUint64ByQ64x64(5, 2**64 / 2) // returns 46116860184273879040
```

### Dividing Q Numbers

If we compute 1 ÷ 1 we expect the result to be 1. Suppose we are using Q64.64. “1” is $2^{64}$. If we compute $2^{64}\div2^{64}$ we get 1 as the result, not $2^{64}$. To correct this, we could leftshift the result by `n` bits, but this violates the principle of “multiply before divide to avoid precision loss.” Therefore, the correct way to divide two Q numbers is to first left-shift the numerator, then do the division: 

```solidity
function divQ64x64ByQ64x64(uint128 x, uint128 y) public pure returns (uint128) {

    return uint128(uint256(x) << 64 / uint256(y));
}

divQ64x64ByQ64x64(5, 2**64 / 2) // returns 46116860184273879040
```

## Summary

- Q numbers are a design pattern to hold fractional numbers in Ethereum.
- They are more efficient than Ethereum decimal representation since multiplication and division can be accomplished with bitshifting.
- A Q number is represented as Qm.n where `m` is the number of bits for the integer and `n` is the number of bits for the fraction.
- Each bit after the decimal represents the value 1/2, 1/4, 1/8, … etc.
- An integer can be converted into a Q number by doing a left bitshift for `n` bits. Q number can be converted to an integer by truncating the fraction bits, or rightshifting by `n` bits.
- If we divide a Q number by $2^{n}$ in a language that supports floating points, we can see the intended fractional representation.
- Given two integers `a` and `b`, ensure `a` fits within `m` bits. Compute their ratio as a Qm.n number via `a << n / b`. The number of bits used to hold the resulting fixed-point number must be the number of bits of `a` (`m`) plus the `n`, i.e. Qm.n.
- Q numbers can be added together “as is” as long as the decimals are aligned.
- If we multiply two Q numbers together, we need to leftshift the result by `n` so that the result has `n` decimals.
- If we divide two Q numbers together, we need to first leftshift the numerator by `n`.
- Both division and multiplication need to be careful to avoid temporary overflow.
