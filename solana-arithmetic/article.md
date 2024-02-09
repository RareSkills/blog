# Day 2: Function Arguments, Math, and Arithmetic Overflow ✅ ✅ ✅

Today we will learn how to create a Solana program that accomplishes the same things as the Solidity contract below. We will also learn how Solana handles arithmetic issues like overflow.

```solidity
contract Day2 {

	event Result(uint256);
	event Who(string, address);
	
	function doSomeMath(uint256 a, uint256 b) public {
		require(b != 0, "divide by zero");
		uint256 result = a / b;
		emit Result(result);
	}

	function sayHelloToMe() public {
		emit Who("Hello World", msg.sender);
	}
}
```

Let’s start a new project

```
anchor init day2
cd day2
anchor build

# replace the program id in programs/day2/src/lib.rs and anchor.toml
anchor keys sync
```

Be sure you have the Solana test validator running in one terminal:

```solidity
solana-test-validator
```

And the Solana logs in another:

```solidity
solana logs
```

Make sure the newly scaffolded program works properly by running the tests

```solidity
anchor test --skip-local-validator
```

## Supplying Function Arguments

Before we do any math, let’s change the initialize function to receive two integers. Ethereum uses uint256 as the “standard” integer size. On Solana, it is u64 — this is equivalent to uint64 in Solidity.

### Passing unsigned integers

The default initialize function will look like the following:

```rust
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    Ok(())
}
```

Modify initialize() function in lib.rs to be as follows.

```rust
pub fn initialize(ctx: Context<Initialize>, a: u64, b: u64) -> Result<()> {
    msg!("You sent {} and {}", a, b);
    Ok(())
}
```

Now we need to change the test in `./tests/day2.ts`

```tsx
it("Is initialized!", async () => {
  // Add your test here.
  const tx = await program.methods.initialize(new anchor.BN(777), new anchor.BN(888)).rpc();
  console.log("Your transaction signature", tx);
});
```

Now re-run `anchor test --skip-local-validator`.

When we look in the logs, we should see something like the following

![Screenshot 2023-05-11 at 1.33.53 PM.png](Day%202%20Function%20Arguments,%20Math,%20and%20Arithmetic%20Ove%202406a3d0ec1e43fe9070c997a8e539bf/Screenshot_2023-05-11_at_1.33.53_PM.png)

### Passing a string

Now let’s illustrate how to pass a string as an argument.

```rust
pub fn initialize(ctx: Context<Initialize>, a: u64, b: u64, message: String) -> Result<()> {
    msg!("You said {:?}", message);
    msg!("You sent {} and {}", a, b);
    Ok(())
}
```

And change the test

```tsx
it("Is initialized!", async () => {
    // Add your test here.
    const tx = await program.methods.initialize(new anchor.BN(777), new anchor.BN(888), "hello").rpc();
    console.log("Your transaction signature", tx);
  });
```

When we run the test, we see the new log

![Screenshot 2023-05-11 at 1.41.09 PM.png](Day%202%20Function%20Arguments,%20Math,%20and%20Arithmetic%20Ove%202406a3d0ec1e43fe9070c997a8e539bf/Screenshot_2023-05-11_at_1.41.09_PM.png)

### Array of numbers

Next we add a function (and test) to illustrate passing an array of numbers. In Rust, a “vector”, or `Vec` is what Solidity calls an “array.”

```rust
pub fn initialize(ctx: Context<Initialize>, a: u64, b: u64, message: String) -> Result<()> {
    msg!("You said {:?}", message);
    msg!("You sent {} and {}", a, b);
    Ok(())
}

// added this function
pub fn array(ctx: Context<Initialize>, arr: Vec<u64>) -> Result<()> {
    msg!("Your array {:?}", arr);
    Ok(())
}
```

And we update the unit test as follows

```tsx
it("Is initialized!", async () => {
  // Add your test here.
  const tx = await program.methods.initialize(new anchor.BN(777), new anchor.BN(888), "hello").rpc();
  console.log("Your transaction signature", tx);
});

// added this test
it("Array test", async () => {
  const tx = await program.methods.array([new anchor.BN(777), new anchor.BN(888)]).rpc();
  console.log("Your transaction signature", tx);
});
```

And we run the tests again and view the logs to see the array output:

![Screenshot 2023-05-11 at 1.43.04 PM.png](Day%202%20Function%20Arguments,%20Math,%20and%20Arithmetic%20Ove%202406a3d0ec1e43fe9070c997a8e539bf/Screenshot_2023-05-11_at_1.43.04_PM.png)

**Tip:** If you are stuck with an issue on your Anchor tests, try googling for “[Solana web3 js](https://solana-labs.github.io/solana-web3.js/)” as it relates to your error. The Typescript library used by Anchor is the Solana web3 js library.

## Math in Solana

### Floating point math

Solana has some, although limited, native support for floating point operations. However, it’s best to avoid floating point operations because of how computationally intensive they are (we will see an example of this later). Note that Solidity has *no* native support for floating point operations.

Read more about the limitations of using floats [here](https://docs.solana.com/developing/on-chain-programs/limitations#float-rust-types-support).

## Arithmetic Overflow

Arithmetic overflow was a common attack vector in Solidity until version 0.8.0 built overflow protection into the language by default. In Solidity 0.8.0 or higher, the overflow checks are done by default. Since these checks consume gas, sometimes devs strategically disable them with the “unchecked” block.

## How does Solana defend against arithmetic overflow?

### Method 1: `overflow-checks = true` in Cargo.toml

If the key `overflow-checks` is set to `true` in the `Cargo.toml` file, then Rust will add overflow checks at the compiler level. See the screenshot of `Cargo.toml` next

![Screenshot 2024-01-10 at 3.18.36 PM.png](Day%202%20Function%20Arguments,%20Math,%20and%20Arithmetic%20Ove%202406a3d0ec1e43fe9070c997a8e539bf/Screenshot_2024-01-10_at_3.18.36_PM.png)

If the Cargo.toml file is configured in this manner, you don’t need to worry about overflow.

However, adding overflow checks increases the compute cost of the transaction (we will revisit this shortly). So under some circumstances where compute cost is an issue, you may wish to set `overflow-checks` to false. To strategically check for overflows, you can use the Rust `checked_*` operators in Rust.

### Method 2: using `checked_*` operators.

Let’s look at how overflow checks are applied to arithmetic operations within Rust itself. Consider the snippet of Rust below.

- On line 1, we do arithmetic using the usual `+` operator, which overflows silently.
- On line 2, we use `.checked_add`, which will throw an error if an overflow happens. Note that we have `.checked_*` available for other operations, like `checked_sub` and `checked_mul`.

```rust
let x: u64 = y + z; // will silently overflow
let xSafe: u64 = y.checked_add(z).unwrap(); // will panic if overflow happens

// checked_sub, checked_mul, etc are also available
```

**Exercise 1:** Set `overflow-checks = true` create a test case where you underflow a `u64` by doing `0 - 1`. You will need to pass those numbers in as arguments or the code won’t compile. What happens?

You’ll see the transaction fails (with a rather cryptic error message shown below) when the test runs. That’s because Anchor turned on overflow protection. 

![Screenshot 2024-01-06 at 2.11.52 PM.png](Day%202%20Function%20Arguments,%20Math,%20and%20Arithmetic%20Ove%202406a3d0ec1e43fe9070c997a8e539bf/Screenshot_2024-01-06_at_2.11.52_PM.png)

**Exercise 2:** Now change `overflow-checks` to false, then run the test again. You should see an underflow value of `18446744073709551615`.

**Exercise 3:** With overflow protection disabled in `Cargo.toml`, do **let result = a.checked_sub(b).unwrap();** with a = 0 and b = 1. What happens?

Should you just leave `overflow-checks = true` in the `Cargo.toml` file for your Anchor project? Generally, yes. But if you are doing some intensive calculations, you might want to set `overflow-checks` to false and strategically defend against overflows in key junctures to save compute cost, which we will demonstrate next.

## Solana compute units 101

In Ethereum, a transaction runs until it consumes the “gas limit” specified by the transaction. Solana calls “gas” a “compute unit.” By default, a transaction is limited to 200,000 compute units. If more than 200,000 compute units are consumed, the transaction reverts.

### Determining compute costs of a transaction in Solana

Solana is indeed cheap to use compared to Ethereum, but that does not mean your optimizoooor skills in Ethereum development are useless. Let’s measure how much compute units our math functions require.                   

The Solana logs terminal also shows how many compute units were used. We’ve provided benchmarks for the checked and unchecked subtraction below.

With overflow protection disabled consumes 824 compute units.

![Screenshot 2023-05-11 at 2.09.47 PM.png](Day%202%20Function%20Arguments,%20Math,%20and%20Arithmetic%20Ove%202406a3d0ec1e43fe9070c997a8e539bf/Screenshot_2023-05-11_at_2.09.47_PM.png)

With overflow protection enabled in consumes 872 compute units.

![Screenshot 2023-05-11 at 2.11.15 PM.png](Day%202%20Function%20Arguments,%20Math,%20and%20Arithmetic%20Ove%202406a3d0ec1e43fe9070c997a8e539bf/Screenshot_2023-05-11_at_2.11.15_PM.png)

As you can see, just doing a simple math operation takes up almost 1000 units. Since we have 200k units, we can only do a few hundred simple arithmetic operations within the per-transaction gas limit.  So, while transactions on Solana are generally cheaper than on Ethereum, we are still limited by a relatively small compute unit cap and would not be able to carry out computationally intensive tasks like fluid dynamic simulations on the Solana chain. 

We’ll revisit transaction cost later.

## Powers does not use the same syntax as Solidity

In Solidity, if we want to raise `x` to the `y` power, we do

```rust
uint256 result = x ** y;
```

Rust does not use this syntax. Instead, it uses `.pow`

```rust
let x: u64 = 2; // it is important that the base's data type is explicit
let y = 3; // the exponent data type can be inferred
let result = x.pow(y);
```

There is also `.checked_pow` if you are concerned about overflow.

## Floating points

One nice thing about using Rust for smart contracts is that we don’t have to import libraries like Solmate or Solady to do math. Rust is a pretty sophisticated language with a lot of operations built in, and if we need some piece of code, we can look outside the Solana ecosystem for a Rust crate (this is what libraries are called in Rust) to do the job.

Let’s take the cube root of 50. The cube root function for floats is built into the Rust language with the function `cbrt()`.

```solidity
// note that we changed `a` to f32 (float 32)
// because `cbrt()` is not available for u64
pub fn initialize(ctx: Context<Initialize>, a: f32) -> Result<()> {
  msg!("You said {:?}", a.cbrt());
  Ok(());
}
```

Remember how we said in an earlier section that floats can be computationally expensive? Well, here we see our cube root operation consumed over 5 times as much as simple arithmetic on unsigned integers.

![Screenshot 2023-05-11 at 3.14.49 PM.png](Day%202%20Function%20Arguments,%20Math,%20and%20Arithmetic%20Ove%202406a3d0ec1e43fe9070c997a8e539bf/Screenshot_2023-05-11_at_3.14.49_PM.png)

**Exercise 4: Build a calculator that does the +, -, x, and ÷. and also sqrt and log10.**