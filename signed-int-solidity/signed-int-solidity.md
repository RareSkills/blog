# Solidity Signed Integer

Solidity signed integers enable using negative numbers in a smart contract. This article documents how they are used at the EVM level. Basic familiarity with the EVM and binary numbers is assumed.

## Two's Complement Explained

### Solidity and the EVM use Two's Complement representation for signed integers

Like every datatype, Solidity still uses 32 byte words to represent signed integers. There isn't any semantic indicator of the type in the EVM, just like there isn't any indicator that a 32 byte slot is actually a boolean, an address, or a 160 bit number. The value is "treated" as negative one at compilation time.

Because you can get the [max value of an integer](https://www.rareskills.io/post/uint-max-value-solidity) with "type(int256).max" or with the .min field to get the minimum. The indicator for whether a number is positive or negative requires an extra bit, so it can only store numbers up to one bit less than the unsigned version.

One's complement means a uint256 becomes a uint255, with the leftmost bit indicating if it is positive or negative. If the EVM used one's complement, this would mean that `type(int256).max == absoluteValue(type(int256.min))` but this is not the case. The maximum magnitude of a Two's Complement negative number is one higher than the maximum magnitude of the positive number. For example, the maximum positive number for int8 is 127, but the maximum magnitude negative number for int8 is -128.

### Patterns and Examples of Two's complement arithmetic.

Rather than going into a bunch of mathematical proofs, let's use some real examples (this is not intended to be proof by example for Two's Complement arithmetic, there is plenty of literature on Two's Complement for the interested reader).

Let's use an int8 to make the examples more readable. The following is in binary, not hex.

```solidity
int8(0) == 0000 0000
type(int8).max == 0111 1111
type(int8).min == 1000 000
```

It is instructive to see the representations for +1 and -1:

```solidity
int8(1) == 0000 0001
int8(-1) == 1111 1111
```

Let's count downward in two's complement so that a pattern becomes apparent:

```solidity
int8(-2) == 1111 1110
int8(-3) == 1111 1101
int8(-4) == 1111 1100
int8(-5) == 1111 1011
```

You can roughly think of two's complement negative numbers as "counting down." 

Here's the interesting feature of two's complement. -2 + -2 should equal -4, and adding in two's complement and allowing overflow enables this. Here is adding -2 to itself in Python using the two's complement representation

```python
>>> (int(b'11111110', 2) + int(b'11111110', 2) ) % 256
252
>>> bin(252)
'0b11111100'
```

This matches the expected pattern above.

What if we add +4 to -2? We should get +2. Let's see that in action

```python
>>> # -2 + 4
>>> (int(b'11111110', 2) + int(b'00000100', 2)) % 256
>>> 2
```

This only works if both numbers are in two's complement representation. Solidity does not allow you to add unsigned and signed integers together since it is ambiguous what the intent is.

Two's Complement also works with multiplication. The expected result of -2 and -2 is +4, and the reader is encouraged to copy the code from earlier to verify this.

### This does not work for all arithmetic operations.

Two's Complement does not require changes to addition, subtraction, multiplication, or even a left bitshift (<<). These correspond to the EVM op codes ADD, SUB, MUL, and SHL. We will discuss why shift left still works in two's complement in a later section in this tutorial.

However, multiplication, modulo, right shift, and casting to a larger signed integer cannot be done using the signed methods and require their own op codes. Similarly, the traditional comparison operators won't work, as negative numbers "appear" to be larger than positive ones.

## Ethereum op codes for signed arithmetic

### sdiv  

#### Gas cost: 5   
SDIV, or signed division, is for dividing signed numbers. This opcode is used behind the scenes in code like the following.

```solidity
function divide(int256 a, int256 b) public pure returns (int256 quotient) 
{
    quotient = a / b;
}
```

### smod

#### Gas cost: 5
Since two's complement arithmetic needs it's own opcode for div, it's no surprise the same applies to take the modulus (remainder).

```solidity
function divide(int256 a, int256 b) public pure returns (int256 remainder) 
{
    remainder = a % b;
}
```

### slt and sgt

#### Gas cost: 3
To compare the magnitude of signed numbers, we first need to determine if it is positive or negative, then compare the magnitude. These op codes do that operation in one step.

Like the unsigned counterparts, it is more gas efficient to avoid the >= and <= where possible and use the strict inequality operators instead.

### sar - signed arithmetic shift right

#### Gas cost: 3
SAR is a very rarely used op code, but it will appear on the compilation result of this solidity code. Note that x is an integer and y is an unsigned integer.

```solidity
contract SarExample {
    function main(int256 x, uint256 y) public pure returns (int256 res) {
        res = x >> y;
    }
}
```

How do we make sense of this? In terms of ordinary unsigned numbers, shifting the bits right by one has the effect of dividing by two, shifting by two has the effect of dividing by four, etc.

```solidity
uint256 x = 8 >> 2; // x = 2
uint256 y = 4 >> 1; // y = 2
```

If you do this to a signed integer, this phenomena is preserved.

```solidity
int256 x = -8 >> 2; // x = -2
int256 y = -4 >> 1; // y = -2
```

Why isn't there a SAL (signed arithmetic left shift) op code? What should happen in the following example?

```solidity
int256 x = -8 << 2; // x = -32
int256 y = -4 << 1; // y = -8
```

We multiplied by 4 and 2, respectively. In Two's Complement, shifting left preserves the number as expected.

Under the hood, a regular SHL (shift left) opcode was used. There is no need for a special case for arithmetic left shift. This might seem unintuitive, as the number gets larger as rightmost bits are zeroed out. But remember, the maximum negative value in Two's Complement is when the leftmost bit is one, and all the other bits are zero.

### signextend evm

#### Gas cost: 5   
A signed integer smaller than 256 bits will have leading zeros. However, Two's Complement negative numbers always start with the leftmost bit at one. Therefore, if a Two's Complement integer is cast to a larger type, the value will change from negative to positive since the leftmost bits will be zero. Signextend handles this transition seamlessly.

### signextend solidity

You can't use signextend directly in Solidity, but it's used behind the scenes when a smaller integer is casted to a larger one. The following code contains the signextend opcode in its compiled bytecode to cast the int8 to a int256.

```solidity
contract SignExtendExample {
    function main(int8 x) public pure returns (int256 res) {
        res = x;
    }
}
```

It should be obvious at this point that larger integers cannot be cast to smaller ones.

## Learn more

Learn more advanced topics in our expert [solidity training course](https://www.rareskills.io/solidity-bootcamp).

*Originally Published Apr 11, 2023*
