# Generate Ethereum Address from Private Key Python

## Generate Ethereum Address from Public Key

An Ethereum address is the last 20 bytes of the keccak256 of the public key. The public key algorithm is secp256k1, the same used in Bitcoin.

Because it is an elliptic curve algorithm, the public key is an (x, y) pair that corresponds to a point on the elliptic curve.

## Generate the Elliptic Curve Public Key

The public key is the concatenation of x and y, and that is what we take the hash of.

The code is provided below.

```python
from ecpy.curves import Curve
from ecpy.keys import ECPublicKey, ECPrivateKey
from sha3 import keccak_256

private_key = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

cv = Curve.get_curve('secp256k1')
pv_key = ECPrivateKey(private_key, cv)
pu_key = pv_key.get_public_key()

# equivalent alternative for illustration:
# concat_x_y = bytes.fromhex(hex(pu_key.W.x)[2:] + hex(pu_key.W.y)[2:])

concat_x_y = pu_key.W.x.to_bytes(32, byteorder='big') + pu_key.W.y.to_bytes(32, byteorder='big')
eth_addr = '0x' + keccak_256(concat_x_y).digest()[-20:].hex()

print('private key: ', hex(private_key))
print('eth_address: ', eth_addr)
```

The ecpy library is here [https://github.com/cslashm/ECPy](https://github.com/cslashm/ECPy). This library implements the elliptic curve math in Python, so it won't be as fast as a wrapper around the Bitcoin C implementation, which is used by the [coincurve](https://github.com/ofek/coincurve/) library.

However, the Python implementation allows you to see step by step the elliptic curve math used to derive the public key.

You can use this code to generate an Ethereum vanity address with brute force, but be mindful that if your source of randomness is not secure or has too few bits of randomness, you may fall victim to a hack similar to [this](https://cointelegraph.com/news/almost-1m-in-crypto-stolen-from-vanity-address-exploit).

## Generate private key with Python and coin flips or dice

You can generate an Ethereum address from a private key yourself by flipping a coin 256 times and writing to a string a 1 if it's heads and 0 if it is tails.

Let's say you get the following outcome

```
result = b'1100001011001101010001001100101000001111101101111011001000110001101100011101101011010001011000101111100110010101001001101110111011001000100001010101111100001100100110010010111110110100000010011111100000110101001110000101100101011111001101010001100001000'
```

You can convert that to a private key using the following Python code

```python
# 2 means base 2 for binary

private_key = hex(int(result, 2))

# private key is 0x1859a89941f6f646363b5a2c5f32a4ddd910abe19325f6813f06a70b2be6a308
```

Then, plug that private key into the code from the above section and you've generated your address with your own randomness.

The same thing can be accomplished faster by rolling a 16-sided dice 64 times and writing out the hex string that is produced character by character. Be mindful that most dice don't have a representation for the number zero, so you'll have to subtract 1 from each result.

If you only have traditional six-sided dice, you can write out a string in base 6 (don't forget to subtract 1 from each roll) and do a base conversion to binary. You'll need to keep rolling until you have at least 256 bits for your private key. If you are particularly paranoid about randomness, you can use casino grade dice.

Be cautious using the built-in random number library for Python. It's not intended to be cryptographically secure. We recommend familiarizing yourself with [cryptographically secure randomness](https://www.google.com/search?q=cryptographically+secure+randomness) if you are new to the topic.

Generally, you cannot initialize a hardware wallet using this method because the 24 word recovery phrase they use is not the same thing as a private key used to sign transactions. The 24 word recovery phrase is used to derive multiple private keys for different types of crypto the wallet holds.

## Math: About secp256k1: Public Key from Private Key

### The shape of the Curve

Secp256k1 defines the shape of the elliptic curve $y^2 = x^3 + 7$ (mod p) where p is the prime number `115792089237316195423570985008687907853269984665640564039457584007908834671663`

Or $2^{256} – 2^{32} – 977$

The prime *p* should not be confused with the *order* of the curve. Not every value $0 < n < p$ satisfies the equation above. However, operations on the curve are guaranteed to be *closed*. That is, if two valid points are added or multiplied, the result will be a valid number on the curve.

### The starting point

The other important parameter in secp256k1 is the starting point G. Since G is a point on the elliptic curve, it is 2-dimensional and has the parameters

```python
x = 55066263022277343669578718895168534326250603453777594175500187360389116729240

y = 32670510020758816978083085130507043184471273380659243275938904335757337482424
```

To see G is a valid point, we can plug the numbers into Python

```python
x = 55066263022277343669578718895168534326250603453777594175500187360389116729240
y = 32670510020758816978083085130507043184471273380659243275938904335757337482424
p = 115792089237316195423570985008687907853269984665640564039457584007908834671663

assert pow(y, 2, p) == (pow(x, 3) + 7) % p
```

To create a public / private key pair, a random number *s* is created (this is the secret key). The point G is added to itself *s* times and the new point (x, y) is the public key. It is not feasible to derive *s* from G and (x, y) if *s* is sufficiently large.

Signing messages in this scheme boils down to demonstrating you know *s* without revealing it.

Adding G to itself s times is the same as multiplying s * G. In fact, we can see this operation at a lower level in by stripping away some of the abstractions the library is providing.

```python
from ecpy.curves import Curve
from sha3 import keccak_256

private_key = 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80

cv     = Curve.get_curve('secp256k1')
pu_key = private_key * cv.generator # just multiplying the private key by generator point (EC multiplication)

concat_x_y = pu_key.x.to_bytes(32, byteorder='big') + pu_key.y.to_bytes(32, byteorder='big')
eth_addr = '0x' + keccak_256(concat_x_y).digest()[-20:].hex()

print('private key: ', hex(private_key))
print('eth_address: ', eth_addr)
```

The public key is simply the private key multiplied by the point G on the secp256k1 elliptic curve. That's it.

## Learn More

Check out our advanced [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) today and become a blockchain developer who knows the hard stuff other coders don't.

*Originally Published Jan 30, 2023*
