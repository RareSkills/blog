# Zero knowledge programming languages

![Zero knowledge proof moon math](https://static.wixstatic.com/media/935a00_dca926dacdd24cdea8a84fa35d9740bd~mv2.webp/v1/fill/w_740,h_725,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_dca926dacdd24cdea8a84fa35d9740bd~mv2.webp)
Zero knowledge proof moon math

[Zero knowledge proofs](https://www.rareskills.io/zk-bootcamp) demonstrate you executed a computation correctly without revealing the inputs to the computation. A zero knowledge programming language is a convenient way to represent that computation.

A common, but misleading analogy for zero knowledge proofs is “I can prove I’m over 21 without showing you my ID card.” Formulated this way, you cannot create a zero knowledge proof.

Zero knowledge proofs don’t prove **_knowledge._** They prove **correct computation** for the *public output of a public algorithm* without revealing the inputs to the computation.

So zero knowledge ID cards don’t work that way. Let’s use a real example. It is possible to take a known number, say 1,093,073, and create a proof that you multiplied two numbers to obtain it without revealing those two numbers (in this case 1103, 991).

The zero knowledge proof for this would have three components:

- two numbers (we call them **fields**), in this case, encrypted versions 1103 and 991
- a large byte string (the proof)
- a programmatic description of the multiplication algorithm

It is a bit misleading to call the inputs “encrypted” because there isn’t a natural way to decrypt them, but it’s a reasonable way to think about it. You can do mathematically sensible things with the inputs, but you cannot determine what their original values were).

You can think of zero knowledge proofs as a highly generalized version of digital signatures.

With digital signatures, you are proving knowledge of a private key without sharing the private key. You prove this knowledge by first making a message and a public key available. Then you show a signature that is consistent with the message and the public key.

The analogy for zero knowledge proofs works like this:

(public output, encrypted input, zero knowledge proof) —> digital signature

(algorithm) —> public key.

You produce a proof (the byte string) that is consistent with the output and the algorithm. The verifier cannot actually execute the computation, but they can verify

- the algorithm
- the output
- the encrypted inputs, and
- the proof string

are all consistent with each other.

In digital signatures, the public address is known in advance. In zero knowledge applications, the algorithm you are executing is known in advance. Digital signatures change depending on the message you are signing. Likewise, in zero knowledge proofs, your proof string will change with the inputs and outputs you are proving.

In general form, a zero knowledge computation looks like this:

```python
prove(inputs, algorithm_description) -> (encrypted_inputs, proof, public_output)
verify(encrypted_inputs, proof, algorithm_description) == public_output
```

A more interesting example is moving money around in a cryptocurrency like zcash. Each transaction to move money is a state transition, which itself is a computation. The public output is the total supply (which doesn’t change modulo miner rewards). The encrypted input is the balance transfer between addresses.

Since zero knowledge proofs are just computations, they very naturally lend themselves to being described with programming languages. Remember the programmatic description of the multiplication algorithm from earlier? That can be conveniently represented in a traditional programing language that will compile it into “moon math” behind the scenes.

Without further ado, here is the list of zero knowledge programming languages.

## [Circom](https://docs.circom.io/), produced by [iden3](https://iden3.io/)

If you come from an electrical engineering background, rejoice. The dominant way to represent computation in zero knowledge is with boolean circuits. Boolean circuits are really gigantic math equations, so they are easily converted to the moon math equations that happen behind the scenes.

You may be wondering, if zero knowledge is represented with circuits, how can something like a hash algorithm or for loop be carried out? This requires several steps, whereas a circuit computes things in one go.

This is accomplished by treating each step as a separate circuit and having a list of proofs for each step. This is fortunate for us, because describing most computations in terms of circuits is very challenging.

Thankfully, rather than having to implement ALU (arithmetic logic units) ourselves, Circom provides a layer of abstraction on top of it. Here is code for adding two numbers together

```javascript
pragma circom 2.0.0;

template Add(){
    //Declaration of signals
    signal input in1;
    signal input in2;
    signal output out;
    out <== in1 + in2;
}

component main {public [in1,in2]} = Add();
```

These templates allow you to assembly building blocks together to do much more complicated operations, such as concatenate byte strings together, elliptic curve arithmetic, or hash functions.

## [Zokrates](https://zokrates.github.io/)

If six lines of code to add two numbers together is a turn off for you, thankfully there are higher level abstractions for this. Here is how Zokrates does it.

```python
def main(field x, field y) -> field {
    field z = x + y;
    return z;
}
```

Why do zero knowledge programming languages refer to numbers as fields? Mathematics in zero knowledge proofs is always done modulo a large prime number (as in most cryptographic algorithms). The term “field” here is shorthand for “[finite field](https://en.wikipedia.org/wiki/Finite_field)” since the number of possible values the number can take on is finite. Specifically, it is the size of the prime number you are doing modulo arithmetic over.

Obviously, proving you know the sum of two numbers without revealing those numbers is not very interesting. Here is a better application using the example from the beginning.

### Zero knowledge ID card in Zokrates

![zero knowledge proof id card algorithm](https://static.wixstatic.com/media/935a00_3bfce68347c24e8bb6ef7733ef242a81~mv2.webp/v1/fill/w_740,h_740,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_3bfce68347c24e8bb6ef7733ef242a81~mv2.webp)
zero knowledge proof id card algorithm

With some modifications, we can make our zero knowledge ID card work.

To begin, it isn’t possible to demonstrate age without some authority verifying you were born on a certain date. But here’s a way you can turn knowledge of a fact into a proof of a computation.

The government takes your birthday, your citizen id number, and a random salt. They concatenate these strings together, hash them, and digitally sign the hash digest. When you go to an establishment to order something that requires proving you are over 21, you would provide your encrypted birthday, your encrypted citizen id, your encrypted random salt, the hash the government signed, and the byte string proof.

The goal is to prove that you can correctly carry out the algorithm to produce the signed hash.

Normally, you can’t concatenate encrypted values together and recover the signed hash, but that’s where zero knowledge programming comes in. You aren’t actually hashing the encrypted inputs, you are proving you carried out a computation correctly without revealing the inputs.

In Zokrates, this looks surprisingly ergonomic

```python
import "hashes/sha256/512bitPacked" as sha256packed;

def main(private field birthday,
         private field citizenId,
         private field salt,
         public requiredBirthday,
         public field hashFirst128bits,
         public field hashSecond128bits) {

    field[2] hash = sha256packed([birthday, citizenId, salt]);
    assert(hash[0] == hashFirst128bits);
    assert(hash[1] == hashSecond128bits);
    assert(birthday < requiredBirthday);
    return;
}

// the signature for the hash would be validated in a traditional way
```

This doesn’t look so scary, does it!? We are ensuring the secret values hash to the expected hash and making sure the input corresponding to birthday hits the threshold.

One thing that jumps out: how can the line

```python
assert(birthday < requiredBirthday)
```

be executed if birthday is encrypted?

The code is compiled to math where this actually meaningful — it isn’t “executed” in the way we normally think about it. Similarly, the hash circuit is not actually hashing the inputs. It’s just verifying that the encrypted inputs are consistent with the public hash output as a function of the underlying circuit and accompanying proof.

Original whitepaper for Zokrates, published at IEEE 2018 ([http://www.ise.tu-berlin.de/fileadmin/fg308/publications/2018/2018_eberhardt_ZoKrates.pdf](http://www.ise.tu-berlin.de/fileadmin/fg308/publications/2018/2018_eberhardt_ZoKrates.pdf))

## [Leo](https://leo-lang.org/)

The [Aleo blockchain](https://www.aleo.org/) is a privacy focused smart contract chain, with Leo as its primary programming language. You can see what the language is like in their [online playground](https://play.leo-lang.org/).

At the time of writing, only the testnet has launched.

Here is the language adding two numbers together.

```rust
program helloworld.aleo {
  transition main(public a: u32, b: u32) -> u32 {
      let c: u32 = a + b;
      return c;
  }
}
```

Leo whitepaper / academic paper: [https://eprint.iacr.org/2021/651.pdf](https://eprint.iacr.org/2021/651.pdf)

## Layer Two Applications

Although zero knowledge proofs are famous for privacy, they have a powerful use case for blockchain scalability. Remember, the verifier can’t actually hash or add the encrypted inputs together and obtain sensible results. They just validate that the inputs, outputs, circuit, and proofs are consistent.

Interestingly enough, proving consistency is computationally easier than carrying out the actual computation. This means the block producer assembling the transaction has a fair amount of computational burden because they must carry out the computation and produce the proofs, ***but the other validators who are validating the proposed block have a lot less work to do because they only have to validate the proof, which is asymptotically easier.***

Consensus doesn’t happen when the block is produced, it happens when over two thirds of the network agrees that the block is valid. The heavier the computation they have to carry out, the longer consensus takes, and this causes throughput bottlenecks.

Because this use case optimizes for fast verification, you shouldn’t assume that the operations are private by default. The implementations of the following languages do not necessarily optimize for preserving privacy unless they explicitly say so.

## [Noir](https://aztec.network/noir/) by [Aztec](https://aztec.network/)

Noir is the language for Aztec, a privacy focused L2 for [Ethereum](https://www.rareskills.io/post/generate-ethereum-address-from-private-key-python). The syntax is heavily inspired by Rust; even the build tool is called “nargo”, an obvious nod to Rust’s cargo.

Here is Noir adding two numbers together

```rust
fn foo(x : Field, y : Field) -> Field {
    x + y
}
```

In Rust, and in Noir, statements that do not have a semicolon are interpreted as function returns, in case you were wondering why the return statement disappeared.

## [Cairo](https://www.cairo-lang.org/) by [Starkware](https://starkware.co/)

Starknet is another L2.

The name is a portmanteau of CPU Algebraic Intermediate Representation. Intermediate representation languages are intended to be just “barely above the assembly” but in zero knowledge proofs, the “assembly” are the algebra formulas for proving zero knowledge.

C, or C++ developers (or Solidity developers comfortable with assembly) will feel at home with this language since it is fairly low level. Here is Cairo’s [playground](https://www.cairo-lang.org/playground/).

In keeping with tradition, Cairo refers to numbers as “fields” but calls them “field elements” and that gets condensed to “felt” in the syntax (**f**ield + **el**emen**t**).

Here is Cairo adding two numbers

```go
%builtins output

// Import the serialize_word() function.
from starkware.cairo.common.serialize import serialize_word

func main{output_ptr: felt*}() {
    tempvar x = 12;
    tempvar y = 14;
    tempvar z = x + y;
    serialize_word(z);
    return ();
}
```

Cairo whitepaper: [https://eprint.iacr.org/2021/1063.pdf](https://eprint.iacr.org/2021/1063.pdf)

## Solidity

Didn’t expect to see this here did you?!

You may have heard some hype around “zk evms” that [Polygon](https://polygon.technology/solutions/polygon-zkevm/) and [Matter Labs](https://matter-labs.io/) are producing. The hype is that you can compile vanilla solidity into regular EVM bytecode and also produce a zero knowledge proof that you executed the bytecode correctly. This of course requires a specialized virtual machine, which the companies tend to brand as some variation of “zk evm”.

You can think of the EVM’s state as represented by two arrays: the stack and memory. Each operation alters the stack and/or the memory. Each step is a proof that the way you altered the state is consistent with the opcode you just executed and the previous state.

Because of this innovation Solidity is likely to stay relevant for a long time, so check out our [Solidity bootcamp](https://www.rareskills.io/solidity-bootcamp)[!](https://www.rareskills.io/solidity-bootcamp) It’s a prerequisite to our [zero knowledge bootcamp](https://www.rareskills.io/zk-bootcamp).

## Which one should I use, and further resources

If you plan on developing on Aleo, Starknet, or Aztec, you’ll have to use their respective languages. But if you are trying to develop a smart contract on the Ethereum that is privacy focused, you’ll need to use either Circom or Zokrates. Noir currently has limited support for compiling to solidity also.

While it may be tempting to jump straight into using a language friendlier than Circom, it is worth developing an intuition for how circuit programming functions. After all, the languages here ultimately compile to something similar to it. You are going to notice odd constraints in the languages subsequent to Circom in this list that won’t make sense unless you have an idea of what is going on under the hood. For example, all branches of “if” statements execute in Zokrates, and memory cannot be overwritten in Cairo.

There is no such thing as a gentle introduction to the math behind zero knowledge proofs, but [this paper](https://arxiv.org/pdf/1906.07221.pdf) should be understandable by anyone who has taken precalculus or a basic cryptography course.

## Practice Zero Knowledge Puzzles

If you want a gentle introduction to Circom, read their documentation and try solving our open source [Zero Knowledge Puzzles](https://github.com/RareSkills/zero-knowledge-puzzles).

*Originally Published Dec 22, 2022*
