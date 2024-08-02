# Circom language tutorial with circomlib walkthrough

This tutorial introduces the Circom language and how to use it, along with common pitfalls. We will also explain a significant portion of the circomlib library in order to introduce common design patterns.

## A note about production use
Circom is a fantastic tool for learning ZK-SNARKS. However, because it is quite low-level, there are more opportunities to accidentally add subtle bugs. In real applications, programmers should consider using higher [level zero knowledge programming languages](https://www.rareskills.io/post/zero-knowledge-programming-language). You should always get an audit before deploying a smart contract that holds user funds, but this is especially true for [ZK circuits](rareskills.io/post/arithmetic-circuit), as the attack vectors are less well known.

## Prerequisites
Although it is possible to program in Circom without understanding what a [Rank 1 Constraint System](https://www.rareskills.io/post/rank-1-constraint-system) is, you will have a far easier time developing a mental model of the language if you do. Circom is essentially an ergonomic wrapper for creating Rank 1 Constraint Systems.

## Installation
Follow the steps here to [install circom](https://iden3.github.io/circom/getting-started/installation/)

You will also need to install [snarkjs](https://github.com/iden3/snarkjs)

```bash
npm install -g snarkjs@latest
```

## Hello World
The hello world of ZK-circuits is doing a multiplication, so let's start with that.

```javascript
pragma circom 2.1.6;

template Multiply() {
  signal input a;
  signal input b;
  signal output out;

  out <== a * b;
}

component main = Multiply();
```

Save the above code as `multiply.circom` and in the terminal run

```bash
circom multiply.circom
template instances: 1
Everything went okay, circom safe
```

to ensure it compiles.

### Generating an R1CS file
To convert the circuit to an R1CS, run the following terminal command:

```bash
circom multiply.circom --r1cs --sym
```

The `--r1cs` flag tells circom to generate an R1CS file and the `--sym` flag means "save the variable names." That will become clear shortly.

Two new files are created:

```
multiply.r1cs
multiply.sym
```

If you open `multiply.r1cs`, you will see a bunch of binary data, but inside the `.sym` file, you will see the names of the variables.

To read the R1CS file, we use snarkjs as follows:

```bash
snarkjs r1cs print multiply.r1cs
```

and we will get the following output

```
[INFO]  snarkJS: [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.a ] * [ main.b ] - [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.out ] = 0
```

Remember, everything is done in a [finite field](https://www.rareskills.io/post/finite-fields) when dealing with arithmetic circuits, so the massive number you see is essentially `-1`. The R1CS equation is equivalent to

```javascript
-1 * a * b - (-1*out) = 0;
```

With a little algebra, we can see this is equivalent to `a * b = out`, which is our original circuit.

The steps of the algebra are as follows:
```
-1 * a * b - (-1*out) = 0;
-1 * a * b = -1*out;
a * b = out;
```

## Non quadratic constraints are not allowed!
A valid R1CS must have exactly one multiplication per constraint (a constraint is a row in R1CS, and `<==` or `===` in Circom). If we try to do two (or more) multiplications, this will fail. All constraints with more than one multiplication need to split into two constraints. Consider the following (non-compiling) example:

```javascript
pragma circom 2.1.6;

template Multiply() {
  signal input a;
  signal input b;
  signal input c;
  signal output out;

  out <== a * b * c;
}

component main = Multiply();
```

When we run `circom multiply.circom` we will get the following error:

```bash
error[T3001]: Non quadratic constraints are not allowed!
  ┌─ "multiply.circom":9:3
  │
9 │   out <== a * b * c;
  │   ^^^^^^^^^^^^^^^^^ found here
  │
  = call trace:
    ->Multiply

previous errors were found
```

A constraint in Circom is represented by the `<==` operator. For this particular constraint, we have two multiplications for one constraint. To solve this, we need to create a separate constraint so that each constraint has only one multiplication.

### Breaking up non-quadratic constraints
Fixing the problem above is straightforward. We introduce an intermediate signal s1 and have it constrain the first multiplication, then combine the output of s1 with the third input. Now we have two constraints and two multiplications, as Circom expects it.

```javascript
pragma circom 2.1.6;

template Multiply() {
  signal input a;
  signal input b;
  signal input c;
  signal s1;
  signal output out;
  
  s1 <== a * b;
  out <== s1 * c;
}

component main = Multiply();
```

When we regenerate and print the R1CS, we get the following

```[INFO]  snarkJS: [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.a ] * [ main.b ] - [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.s1 ] = 0
[INFO]  snarkJS: [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.s1 ] * [ main.c ] - [ 21888242871839275222246405745257275088548364400416034343698204186575808495616main.out ] = 0
```
With a little algebra, this translates to

```
a * b  = s1
s1 * c = out
```
And we can see this is encoding the same computation from our circuit.

## Computing the witness
Run the following command to create code to generate the witness vector

```
circom multiply.circom --r1cs --sym --wasm
```

This will regenerate the R1CS and symbol file, but also create a folder called `multiply_js/`. `cd` to that folder.

Next, we need to create an input.json file in that folder. This is a map from the names of the signals designated input to the value that the prover will supply to them. Let’s set our `input.json` to have the following values:

```json
{"a": "2","b": "3","c": "5"}
```

If we didn't create the `.sym` file from earlier, it would not be possible for Circom to know what input signals we were trying to map to. Because s1 and out are not input signals, we don't create key value pairs for those. The `a`, `b`, and `c` here correspond to the names of the input signals.

```javascript
signal input a;
signal input b;
signal input c;
```

Now we calculate and export the witness with the following command:

```bash
node generate_witness.js multiply.wasm input.json witness.wtns

snarkjs wtns export json witness.wtns

cat witness.json
[
 "1",
 "30",
 "2",
 "3",
 "5",
 "6"
]
```

The computed witness has values `[1, 30, 2, 3, 5, 6]`. 2, 3, and 5 are the inputs, and 6 is the intermediate signal `s1` which is the product of 2 and 3.

This follows the expected layout of the R1CS variables in the form `[1, out, a, b, c, s1]`.

## Public inputs
What if we want to make some of the inputs public? In nullifier schemes for example, we hash the concatenation of two numbers and reveal one of them later.

### Motivation: nullifier schemes
As a quick aside, a nullifier scheme works by concatenating two numbers and then hashing them. They are used in the context where we have a set of hashes, and we want to prove we know the preimage to one of them without revealing which one it is.

If the hashes were simply the hash of one number, then revealing that number would reveal which hash we know the preimage of. However, if we don't reveal any information, we could repeat the action of proving we know the preimage of one of the hashes several times. This would be very problematic if this action involved withdrawing money from a smart contract!

By revealing one of the two hashed numbers, we cannot re-use the preimage, but we also don't reveal which hash we know the preimage of because revealing only one of the two numbers isn't sufficient.

Here's how Circom accomplishes public inputs:

```javascript
template SomePublic() {

    signal input a;
    signal input b;
    signal input c;
    signal v;
    signal output out;
    
    v <== a * b; 
    out <== c * v;
}

component main {public [a, c]} = SomePublic();
```

In the example above, `a` and `c` are public inputs, but `b` remains hidden. Note the public keyword when the main component is instantiated.

## Arrays in Circom
Let's create a component that will compute `n` powers of the input.

It would be rather annoying to have to manually write signal input `n` times, so Circom provides an array type of signals to do this. Here is the code

```javascript
pragma circom 2.1.6;

template Powers(n) {
    signal input a;
    signal output powers[n];
    
    powers[0] <== a;
    for (var i = 1; i < n; i++) {
        powers[i] <==  powers[i - 1] * a;
    }
}
component main = Powers(6);
```

A few new syntactic features are introduced with this example: template arguments and variables.

### Template Arguments
In the example above, we see the template is parameterized with `n`, i.e. `Powers(n)`.

A Rank 1 Constraint System must be fixed and immutable, this means we cannot change the number of rows or columns once defined, and we cannot change the values of the matrices or the witness. That is why the final line has hard-coded argument `Powers(6)`, this size must be fixed.

However, if we wanted to re-use this code later to support a different size circuit, then it is more ergonomic to have the template be able to change its size on the fly. Therefore, components can take arguments to parameterize control flows and data structures, but this must be fixed per circuit.

## Circom Variables
The example above is equivalent to the following

```javascript
pragma circom 2.1.6;

template Powers() {
    signal input a;
    signal output powers[6];
   
    powers[0] <== a;
    powers[1] <== powers[0] * a;
    powers[2] <== powers[1] * a;
    powers[3] <== powers[2] * a;
    powers[4] <== powers[3] * a;
    powers[5] <== powers[4] * a;
    
}
component main = Powers();
```

Although we get an identical R1CS, that code is *ugly*. However, this example usefully illustrates that *any circuit that uses variables in its computation can be rewritten to not include variables*.

**Variables build helper code that exists outside the R1CS. They help define the circuit, but they are not part of the circuit.**

The variable `var i` was simply bookkeeping to track the loop iteration while constructing the circuit, it isn't part of the constraints.

### Signal vs variable
A signal is immutable and intended to be one of the columns of the R1CS. A variable is not part of the R1CS. It is intended for computing values outside the R1CS to help define the R1CS.

It would not be accurate to think of signals as "immutable variables" and variables as "mutable variables" the way some languages assign them. The reason signals are immutable is because the witness entries in an R1CS have a fixed value. A solution vector in an R1CS that changes value does not make sense, as you cannot create a proof for it.

The `<--`, `<==`, and `===` operators are used with signals, not variables. We will explain the unfamiliar operators shortly.

When working with variables, Circom behaves like a normal C-like language. The operators `=`, `==`, `>=`, `<=`, and `!=`, `++`, and `--` behave the way you expect them to. This is why the loop example looks familiar.

The following examples are not allowed:
```javascript
signal a;
a = 2; // using a variable assignment for a signal

var v;
v <-- a + b; // using a signal assignment for a variable is not allowed
```

### `===` vs `<==`
The following circuits are equivalent:
```javascript
pragma circom 2.1.6;

template Multiply() {
    signal input a;
    signal input b;
    signal output c;
    
    c <-- a * b;
    c === a * b;
}

template MultiplySame() {
    signal input a;
    signal input b;
    signal output c;
    
    c <== a * b;
}
```

The `<==` operator computes, then assigns, then adds a constraint. If you only want to constrain, use the `===`.

Typically, you'll use the `===` operator when you are trying to enforce that the `<--` assigned the correct value. You'll see this in action when we look at the `IsZero` template.

But before we get to that, let's look at a real example. Suppose we want the prover to supply both the inputs *and* the output. This is how we would do it using the `===` operator.

```javascript
pragma circom 2.1.6;

template Multiply() {
    signal input a;
    signal input b;
    signal input c;

    c === a * b;
}

component main {public [c]} = Multiply();
```

Circom does not require an output signal to exist, as that is merely syntatic sugar for a public input. Remember, an "input" is merely an entry to the witness vector, so everything is an input from a zero knowledge proof perspective. In the above example, there is no output signal, but this is a perfectly valid circuit with proper constraints.

## Wiring templates together
Circom templates are reusable and composeable as the following example illustrates. Here, Square is a template used by SumOfSquares. Note how inputs a and b are "wired" to the component `Square()`.

```javascript
pragma circom 2.1.6;

template Square() {

    signal input in;
    signal output out;

    out <== in * in;
}

template SumOfSquares() {
    signal input a;
    signal input b;
    signal output out;

    component sq1 = Square();
    component sq2 = Square();

    // wiring the components together
    sq1.in <== a;
    sq2.in <== b;

    out <== sq1.out + sq2.out;
}

component main = SumOfSquares();
```

You can think of the `<==` operator as "wiring" the components together by referencing their inputs or outputs.

### Multiple inputs to a component
The example `Square` above takes a single signal as an input, but if a component takes multiple inputs, it is conventional to specify this as an `array in[n]`. The following component takes two numbers and returns their product:

```javascript
template Mul {

    signal input in[2]; // takes two inputs
    signal output out; // single output
    
    out <== in[0] * in[1];
}
```

Knowing how to wire templates together is a prerequisite for when we bring in a library of circuits, which we will show in an upcoming section.

### Signal naming conventions
It is conventional to name the input signals in or as an `array in[]` and `output` signal(s) to be `out` or `out[]`.

## Unsafe Powers, beware of `<--`
When running into the issue of quadratic constraints, it can be tempting to use the `<--` operator to silence the compiler. The following code compiles, and seems to accomplish the same as our earlier powers example.

```javascript
pragma circom 2.1.6;

template Powers {
    signal input a;
    signal output powers[6];
   
    powers[0] <== a;
    powers[1] <== a * a;
    powers[2] <-- a ** 3;
    powers[3] <-- a ** 4;
    powers[4] <-- a ** 5;
    powers[5] <-- a ** 6;
    
}
component main = Powers();
```

However, when we create the R1CS, we only have one constraint!

```bash
(base) ➜  hello-circom circom bad-powers.circom --r1cs
template instances: 1
non-linear constraints: 1 ### only one constraint ###
linear constraints: 0
public inputs: 0
public outputs: 6
private inputs: 1
private outputs: 0
wires: 7
labels: 8
Written successfully: ./powers.r1cs
Everything went okay, circom safe
```

With only one constraint, the prover only has to set the first element in the array correctly, but can put whatever value they like for the other 5! **You cannot trust proofs that come out of a circuit like this!**

Underconstraints are a major source of security bugs in zero knowledge applications, so triple check that the constraints are actually generated in the R1CS the way you expect them to!

This is why we emphasized understanding Rank 1 Constraint Systems before learning the Circom language, otherwise there is an entire category of bugs that will be hard for you to detect!

You can learn in our other tutorial about how to [exploit underconstrained ZK circuits](https://www.rareskills.io/post/underconstrained-circom).

### When to use `<--`
If the `<--` leads to underconstraints, why would the language include such a foot gun?

When describing algorithms using circuits rather than natural code, the mindset to adopt is "compute, then constrain." Some operations are extremely difficult to model with pure constraints alone. An example of this is shown in the upcoming section.

## Circomlib
Iden3 maintains a repository of useful Circom templates called circomlib. At this point, we have enough requisite Circom knowledge to begin studying these templates, and now is a good time to introduce a simple but useful template that demonstrates the use of `<--`.

### IsZero
The IsZero template returns 1 if the input signal is zero, and zero if the input signal is non-zero.

If you spend some time thinking about how to test if a number is zero using only multiplications, you will find yourself getting stuck; it will turn out to be a fiendishly difficult problem.

Let’s see how the [circomlib component IsZero](https://github.com/iden3/circomlib/blob/master/circuits/comparators.circom#L24-L34) accomplishes this.

```javascript
template IsZero() {
  signal input in;
  signal output out;

  signal inv;

  inv <-- in!=0 ? 1/in : 0;

  out <== -in*inv +1;
  in*out === 0;
}
```

In the example above, `inv` is an auxiliary input signal to make it easier to create a valid circuit. We compute inv to be either zero or the inverse of in outside the R1CS then force inv to be correct as part of the constraints. This is following the pattern of "compute, then constrain."

The signal `inv` is still constrained to either be zero or the modular inverse of `in`.

As an exercise for the reader, draw a truth table showing the following possible combinations

`out`: $\set{1,0}$

`inv`: $\set{0, \text{in}^{-1}, \text{invalid}}$

`in`: $\set{0, \text{non-zero}}$

You will see it is only possible to satisfy the constraints by setting `out` to be 1 when in is 0, and out to 0 when in is non-zero. You cannot monkey around with `inv`, it must follow the rules. This illustrates the pattern of "compute, then constrain."


The arrow creates a constraint and also assigns a value. In the code above, out is constrained to be equal `-in*inv + 1`. However, the final line is not assigning zero to in*out. Rather, it is enforcing that `in*out` does in fact equal zero.

Although you would normally use the ternary operator (or C-like things in general) with variables, you can use the traditional programming syntax with signals if you use the `<--` operator.

**Exercise for the reader:** can you create a template that does the opposite of `IsZero`, without using the `IsZero` template?

## === and <== will not create a constraint if the operation is not quadratic and the optimizer is turned on
Here is quirk of the compiler to look out for

```javascript
template FactorOfFiveFootgun() {

    signal input in;
    signal output out;
    
    out <== in * 5;
}

component main = FactorOfFiveFootgun();
```

Here we are saying we know `x`, such that `5x = out`, where out is public. So if out is 100, we expect `x` to be 20 right? If we look at the R1CS, we see it is in fact empty, no constraints are created!

This is because although `<==` is a constraint, the compiler optimizer eliminates constraints that don't have a multiplication in them.

When we compile the circuit to R1CS, we see it is empty.

```bash
(base) ➜  hello-circom circom footgun.circom --r1cs
template instances: 1
non-linear constraints: 0 ### no constraints ###
linear constraints: 0
public inputs: 0
public outputs: 1
private inputs: 1
private outputs: 0
wires: 2
labels: 3
Written successfully: ./footgun.r1cs
Everything went okay, circom safe
```

However, if we compile the circuit with the optimizer turned off via 

```
circom footgun.circom --r1cs --O0
```

then we see a constraint is created:
```bash
template instances: 1
non-linear constraints: 0
linear constraints: 1 ### Constraint created here
public inputs: 0
private inputs: 1
public outputs: 1
wires: 3
labels: 3
Written successfully: ./footgun.r1cs
Everything went okay
```

**Always sanity check the number of constraints that the compiler creates**.

## Functions in Circom
One thing that makes Circom a bit confusing is that it is both a language for creating arithmetic constraints, *and also* a language for creating constraints programatically *and also* a language for doing regular computations.

Here is an example of separating computation into a function. Note that the "average" here is computed using modular arithmetic, so it won't correspond to the traditional arithmetic mean.

```javascript
pragma circom 2.1.6;

include "node_modules/circomlib/circuits/comparators.circom";

function invert(x) {
    return 1 / x;
}

template Average(n) {

    signal input in[n];
    signal denominator;
    signal output out;

    var sum;
    for (var i = 0; i < n; i++) {
        sum += in[i];
    }

    denominator <-- invert(n);

    component eq = IsEqual();
    eq.in[0] <== denominator;
    eq.in[1] <== n;
    0 === eq.out;

    out <== sum * denominator;
}

component main  = Average(5);
```

## There are constraints depending on the value of the condition and it can be unknown during the constraint generation phase
Here is another easy mistake to make when writing circom code: the signal cannot be an input to an if statement or a loop.

```javascript
template IsOver21() {
    signal input age;
    signal output oldEnough;
    
    if (age >= 21) {
        oldEnough <== 1;
    } else {
        oldEnough <== 0;
    }
}
```

If you try to compile the above code, you will get the error in the header of this section. You will get a similar error if you use a signal to determine the number of times to execute a loop (try it!). The reason this is not allowed, is because we are making the number of constraints in the R1CS a function of one of the R1CS inputs, but this makes no sense.

So how do we check if someone is over 21 years old?

This is actually harder than it seems, since comparing numbers in a finite field is rather tricky.

The Pitfalls of comparing numbers in Circom
If $1 - 2 = 21888242871839275222246405745257275088548364400416034343698204186575808495616$, then is that huge number actually larger than one or less than 1?

You cannot compare numbers in a finite field, as there is no meaning in saying one number is larger than another number.

However, we would still like to be able to compare numbers!


The first requirement before we do any comparisons, is that they must be smaller than the field size, and we will treat any numbers in the field as positive integers, and carefully guard against underflow and overflow.


If you think for a moment about how to represent the following statement

$$a > b$$

using a purely R1CS representation, you’ll get stuck. It’s impossible to do that.

As long as we force numbers to stay within the field order, then we can meaningfully compare them. It isn’t possible to translate the > operator into a set of quadratic constraints.


However, if we convert a field element to a binary representation of the number, then it is possible to make a comparison.

Now we can introduce two more circomlib templates: `Num2Bits` and `LessThan`, with the motivation being to compare integers.

### Num2Bits
The following Circomlib Num2Bits template shows how circom transforms a signal into an array of signals that hold the binary representation.

```javascript
template Num2Bits(n) {
    signal input in;
    signal output out[n];
    var lc1=0; 
    // this serves as an accumulator to "recompute" in bit-by-bit
    var e2=1;
    for (var i = 0; i<n; i++) {
        out[i] <-- (in >> i) & 1;
        out[i] * (out[i] -1 ) === 0; // force out[i] to be 1 or 0
        lc1 += out[i] * e2; //add to the accumulator if the bit is 1 
        e2 = e2+e2; // takes on values 1,2,4,8,...
    }

    lc1 === in;
}
```

Constraining a number to be representable with $n$ bits is equivalent to saying it is less than $2^n$.

### LessThan
How could we compare two numbers that are 9,999 or less without using the normal < and > operators and only using one binary conversion? If we are allowed two binary conversions, this is easy, but it makes for a larger circuit.

It’s a tricky problem, but here’s how circomlib does it.

Let’s say we are comparing $x$ and $y$.

Since 9,999 largest value our inputs can take, we add 10,000 to the $x$ and subtract the $y$ from the sum of $x$ and $y$.

$$z = (10,000 + x) - y$$

No matter what, $z$ will be positive, even if $y$ is 9,999 and $x$ is 0. Since we are dealing with field elements here, we do not want any underflows to happen, and in this case, an underflow cannot occur.

Here is the key part. If $x ≥ y$, then $z$ will between 10,000 and 19,999 and if $x < y$, $z$ will be in the range $[1-9,999]$.

To know if a number is in the range $[10,000-19,999]$, we only need to look at the digit in the 10,000ths place, i.e. 1x,xxx. If we have a $1$ there, then we know $z$ is in the range $[10,000-19,999]$ and if we have something of the form 0x,xxx, then $y$ must have originally been greater than $x$.

Circom just does the same thing in binary rather than decimal representation.

In the decimal analogy, we added 10,000 to a number that is guaranteed to not be larger than 9,999. In the binary case, we add the binary number

$$1\underbrace{000\dots000}_n$$

to a number guaranteed to be at most $n$ bits large. Note 1 left-shifted by $n$ bits, i.e. `1 << n` is:

$$1 << n = 1\underbrace{000\dots000}_n$$

To see why this is the case, consider what $1$ becomes when we compute `1 << 0` and `1 << 1`.

Here is the `LessThan` template in Circom

```javascript
template LessThan(n) {
    assert(n <= 252);
    signal input in[2];
    signal output out;

    component n2b = Num2Bits(n+1);

    // add 1 << n then subtract the number we are comparing
    n2b.in <== in[0] + (1<<n) - in[1];

    // check if the n-th bit is 1 or 0
    out <== 1-n2b.out[n];
}
```

The code is parameterized by $n$ which is the maximum size of the numbers in bits, though there is an enforced upper limit of 252 bits to stay below the limit of the field size to avoid the [alias bug](https://www.rareskills.io/post/circom-aliascheck).

The numbers cannot be larger than $2^n$, so adding `1<<n` ensures the term `in[0] + (1<<n)` will always be larger than `in[1]`. The difference is turned into binary, and if the highest bit is still 1, then `in[0]` is greater than `in[1]`. The final term is 1 minus that bit, so this inverts whether or not the bit is present.

Thus, the component constrains the `out` to be 1 if `in[0]` is less than `in[1]` and 0 otherwise.

### Over21 functional example
Now that we have the requisite toolbox, let’s make an age checker that actually works.

Circomlib provides a comparator called `GreaterThan` which is `LessThan` with a simple twist, so we will not explain it here. The interested reader can consult the source code directly.

To use the circomlib template, create an empty `node_modules/` directory then run

```bash
npm install circomlib
```

Then create the following circuit

```javascript
pragma circom 2.1.6;

include "node_modules/circomlib/circuits/comparators.circom";

template Over21() {

    signal input age;
    signal input ageLimit;
    signal output oldEnough;
    
    // 8 bits is plenty to store age
    component gt = GreaterThan(8);
    gt.in[0] <== age;
    gt.in[1] <== 21;
	0 === gt.out;
    
    oldEnough <== gt.out;
}

component main = Over21();
```

This is the safe way to verify age is is really over 21 (though in a real application, you’d need an authority to attest to age).

### Comparators.circom
The comparators provided in this file are

- `IsZero`
- `IsEqual` (subtracts the two inputs and passes that to IsZero)
- `LessThan`
- `LessEqThan` (derived from LessThan)
- `GreaterThan` (derived from LessThan)
- `GreaterEqThan` (derived from LessThan)
- `ForceEqualIfEnabled`

What does the last one do?

#### ForceEqualIfEnabled
The code for the template ForceEqualIfEnabled is shown below:
```javascript
template ForceEqualIfEnabled() {
    signal input enabled;
    signal input in[2];

    component isz = IsZero();

    in[1] - in[0] ==> isz.in;

    (1 - isz.out)*enabled === 0;
}
```

`ForceEqualIfEnabled` allows us to "turn on and off constraints." It behaves like a conditional constraint, or an if statement. If `enabled` is 0, then it doesn't matter if `in[0]` equals `in[1]` or not; the constraint is effectively ignored because the final constraint will be `0 === 0`. On the other hand, if `enabled` is not zero, then `1 - isz.out` must be zero (i.e. `in[1]` equals `in[0]`) for the constraint to pass.

## Circom assert
Confusingly enough, Circom has an `assert` statement that doesn’t quite do what you would expect.

**The [assert statement does not add any constraints](https://chainsecurity.com/circom-assertions-misconceptions-and-deceptions/).** 

It's merely a safety check so the dev doesn't create circuits with undesirable properties.

For example, we might use it to ensure templates aren’t initialized with length zero arrays if that leads to the possibility of dividing by zero.

You should always assume that a malicious prover can just copy the circuit code, delete the assert statement, and create a proof. That proof will be compatible with the circuit, because the circuit doesn’t incorporate the assert statement.

## Boolean operators
Hopefully you know by now that you cannot use the operators intended for variables on signals. The following code will not compile

```javascript
template And() {

    signal input in[2];
    signal output c;

    c <== in[0] && in[1];
}
```

Remember, signals are field elements, and `&&` is not a valid operator for field elements.

However, if you constrain `a` and `b` to be 0 and 1, you can constrain `c`, `a`, and `b` to follow the behavior of an AND gate.

```javascript
template And() {
    signal input in[2];
    signal output c;
    
    // force inputs to be zero or one
    in[0] === in[0] * in[0];
    in[1] === in[1] * in[1];
    
    // c will be 1 iff in[0] and in[1] are 1
    c <== in[0] * in[1];
}
```

It is a useful exercise for the reader to think of how to accomplish the other boolean operations, NOT, OR, NAND, NOR, XOR, and XNOR. Although every boolean gate can be constructed from a NAND gate, this will make the circuit larger than necessary, so don’t overdo re-using gates.

The solutions are in [circomlib’s gates.circom](https://github.com/iden3/circomlib/blob/master/circuits/gates.circom) file. Note that they do not constrain the input to be zero or one, it is up to the circuit designer to do that.

## ZK-Friendly hash functions
You can imagine that building up a hash function using the method the circuits produced will be huge. Let’s see how big a sha256 hasher that takes 256 bits will be:

```javascript
pragma circom 2.0.0;
include "node_modules/circomlib/circuits/sha256/sha256.circom";

component main = Sha256(256);
```

This innocuous little circuit will produce almost 30,000 constraints:

```bash
(base) ➜  hello-circom circom Sha256-example.circom --r1cs
template instances: 99
non-linear constraints: 29380 ### WOW! ###
linear constraints: 0
public inputs: 0
public outputs: 256
private inputs: 256
private outputs: 0
wires: 29325
labels: 204521
Written successfully: ./Sha256-example.r1cs
Everything went okay, circom safe
```

The reason traditional cryptographic hash functions have so many constraints is that they operate on 32-bit numbers, so field elements must be constrained to "simulate" 32-bit numbers.

This creates a significant amount of work for the prover. As a response, the research community has produced hash functions optimized for circuit representation. In addition to sha256 you will find:

- Mimc
- Pedersen
- Poseidon

ZK-friendly hash functions on the other hand directly use field elements and avoid operations that add a lot of constraints such as bit-shifting or XOR operations.

It is an exercise for the reader to compare the size of the output R1CS of the hash functions. Bear in mind some of the functions take the number of “rounds” as a parameter, which will naturally increase the size of the circuit.

ZK friendly hash functions are a sizable topic that are best suited to have their own discussion, so we defer this to a future article.

## What about the rest?
The remainder of the templates are small enough to be self-explanatory, or are related to zero knowledge digital signature schemes and computing elliptic curves.

The motivation for computing elliptic curves inside a ZK Circuit is that some ZK-friendly hash functions rely on elliptic curves as a core primitive.

This is another large topic which we defer to another article.

Learning Circom from Real World Examples
Two of the classic yet approachable circuits to study are [Tornado Cash](https://www.rareskills.io/post/how-does-tornado-cash-work) and [Semaphore](https://github.com/semaphore-protocol/semaphore/tree/main/packages/circuits).

## Learn more with RareSkills
The best way to learn circom is to use it. Our [Zero Knowledge Puzzles](https://github.com/RareSkills/zero-knowledge-puzzles) provide bite-sized challenges of increasing complexity for you to learn the language. A development environment is provided with unit tests to inform you if you have completed the puzzle successfully. Be aware it is possible to pass the tests with underconstrained circuits, as testing for underconstraint is extremely difficult to do in general. At the very least, you should check the number of constraints generated and make sure the number makes sense.

This material is part of our zero knowledge course. Please see the program to learn more.

*Originally Published September 26, 2023*