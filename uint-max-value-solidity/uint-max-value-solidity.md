# Uint256 max value

The uint256 max value can be obtained with `type(uint256).max;` which is 

```
115792089237316195423570985008687907853269984665640564039457584007913129639935

```

or 2²⁵⁶-1.

But it's cleaner and safer to use `type(uint256).max`.

The same can be used for signed integer types

```
//57896044618658097711785492504343953926634992332820282019728792003956564819967
type(int256).max;

//-57896044618658097711785492504343953926634992332820282019728792003956564819968
type(int256).min;
```

## The math behind maximum values in integers and unsigned integers

For unsigned integers, just put `2 ** n - 1` into the terminal with your favorite interpreted language to get the answer, where `n` is the size of the uint in question, for example `uint128` or `uint32` (or even rarely used but valid sizes like `uint208`). A uintN means there are `N` bits that represent the number, and when all of them are set to 1, this is the maximum binary representation.

The EVM uses [Twos Complement](https://en.wikipedia.org/wiki/Two%27s_complement) to represent signed numbers, so to get the the maximum value of a signed integer, the formula is $2^{n-1} - 1$ and the most negative value is $-2 ^ {n - 1}$.

## Hacky ways to get the uint256 maximum value

You could also specify it with hexadecimal, which would be a little cleaner than using the decimal representation, but still space consuming and error prone.

### Uint256 max value in hex

```
0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
```

That's exactly 64 'f's, don't get it wrong! The 64 is derived from 256 bits divided by 8 to get the number of bytes (32), and each maximum byte is represented in hex by 0xff.

### `~uint256(0)` is equivalent to the uint256 max value

The maximum value of uint256 is 256 bits all set to one. So how about use the bit flip operator? (`~``)

```solidity
type(uint256).max == ~uint256(0) // this will return true
```

Another ugly solution is to underflow the integer

```solidity
function maximum() public pure returns(uint256) {
   unchecked { return uint256(0) - uint256(1); }
}
```

The following code works properly, but it is deceptive and not recommended for use

```solidity
function maxUint() public pure returns (uint256) {
    return 2**256 - 1;
}
```

It returns the correct value, which you can verify with

```solidity
assert(2**256 - 1 == type(uint256).max);
```

If you write `2**256` as a constant in solidity, the code won't compile because the compiler recognized `2**256` is too large to fit in uint256. But if it is immediately followed by a "- 1", then the compiler recognized the result of the arithmetic is valid with the type.

### uint256(-1) doesn't work anymore

In older versions of solidity, `uint256(-1)` could be used to retrieve the maximum value, but that won't compile anymore.

It's better to just avoid all these gymnastics and just do `type(uint256).max`;

## Just how big is a 2^256 - 1?

To put some context on the number, the estimated number of atoms in the known universe is roughly 10^80. To visualize it, let's place the numbers side by side:

```
115792089237316195423570985008687907853269984665640564039457584007913129639935

100000000000000000000000000000000000000000000000000000000000000000000000000000000
```

That roughly means 1,000 uint256 variables could enumerate every atom on the known universe. Since most "things" of interest are made up of far more than 1,000 atoms, a single uint256 can enumerate any useful set of things with plenty of room to spare.

As a corollary, two randomly chosen uint256 values (equivalent to the output of a keccak256 hash), for all practical purposes, won't ever have a collision.

### Learn more

If you are new to Solidity, see our [learn solidity free for beginner course](https://rareskills.io/learn-solidity).

For intermediate and advanced developers please see our expert [solidity bootcamp](https://www.rareskills.io/solidity-bootcamp).

*Originally Published April 2, 2023*