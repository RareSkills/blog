# Solidity Mutation Testing

Mutation testing is a method to check the quality of the test suite by intentionally introducing bugs into the code and ensuring the tests catch the bug.

The kind of bugs that get introduced are straightforward. Consider the following examples:

```solidity
// original function
function mint() external payable {
    require(msg.value >= PRICE, "insufficient msg value");
}

// mutated function
function mint() external public {
    require(msg.value < PRICE, "insufficient msg value");
}
```

In the example above, the inequality operator was flipped. If the unit tests still pass, then the unit tests are simply offering false assurance.

It is important that the bugs be syntactically valid, i.e. still result in compilable solidity code. If the code doesn't compile, then it won't be possible to run the unit tests.

## Line Coverage Without Testing

Let's use the default example [Foundry](https://www.rareskills.io/post/foundry-testing-solidity) provides after running `forge init` and comment out the assert statements.

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/Counter.sol";

contract CounterTest is Test {
    Counter public counter;

    function setUp() public {
        counter = new Counter();
        counter.setNumber(0);
    }

    function testIncrement() public {
        counter.increment();
        //assertEq(counter.number(), 1);
    }

    function testSetNumber(uint256 x) public {
        counter.setNumber(x);
        //assertEq(counter.number(), x);
    }
}
```

If we run [forge coverage](https://www.rareskills.io/post/foundry-forge-coverage), we get the following table:
| File                 | % Lines       | % Statements  | % Branches    | % Funcs       |
|----------------------|---------------|---------------|---------------|---------------|
| script/Counter.s.sol | 0.00% (0/3)   | 0.00% (0/3)   | 100.00% (0/0) | 0.00% (0/2)   |
| src/Counter.sol      | 100.00% (2/2) | 100.00% (2/2) | 100.00% (0/0) | 100.00% (2/2) |
| Total                | 40.00% (2/5)  | 40.00% (2/5)  | 100.00% (0/0) | 50.00% (2/4)  |

Supposedly, we have 100% line and branch coverage on Counter.sol despite having no assert statements! This means we can introduce bugs at will and the tests will still pass.

Now of course, this is a blatant example of what not to do. But it's easy to accidentally make this mistake when optimizing for coverage. Coverage only tells you that you ran the code and it didn't revert. You want to ensure that all the expected state changes are actually taking place (see our other post for more on [solidity unit tesing best practices](https://www.rareskills.io/post/foundry-testing-solidity).

## Kinds of Mutants

Here are some kinds of mutations that may be useful:
- deleting function modifiers
- inverting inequality comparisons
- changing constant values or swapping string constants for empty strings
- replace true with false
- replace `&&` with `||` and bitwise `&` with bitwise `|`
- swapping arithmetic operators (e.g. `+` becomes `-`)
- deleting lines
- swapping lines

## Automatic Mutation Testing

It would be rather tedious to manually mutate the code according to the rules above and then run the test suite. Thus, tools exist that do this automatically. They generate dozens of potential mutations, mutate the code, run the test suite, store the results, and generate a report afterwards. There can be three outcomes:
- mutant survived
- equivalent mutant
- mutant killed

Mutant survived means that the code was changed and the test still passed. An equivalent mutant happens when the bytecode did not change after running the mutation. This can happen if a symbol is randomly replaced with the same symbol, or the mutation doesn't alter the business logic and the compiler optimization ignores the change.

Here is an example of where an equivalent mutant could occur:

```solidity
// before
x = x + 1;
y = y + 1;

// after
y = y + 1;
x = x + 1;
```

Under some circumstances, the compiler might produce the same bytecode after a mutation like this. This is an equivalent mutation. Equivalent mutants might signal unnecessary or dead code like in the following example:

```solidity 
require(false);
// anything that happens here doesn't matter
```

Finally, the mutant killed scenario is the desirable one. It means the code was mutated and the tests failed. Therefore, the tests can actually detect when something goes wrong. If a mutation results in non-compiling code, e.g. deleting a variable declaration that is used later, then the mutant is considered killed.

## 100% Line and Branch Coverage is Important for Mutation Testing

If a line or branch is not covered, then naturally mutating this line will not cause the test to fail.

Consider the following example:

```solidity 
function mint(address to_, string memory questId_) public onlyMinter {
	// business logic
}
```

There is an implied branch here with the `onlyMinter` modifier. If this is only tested in a situation where the minter was the one calling the function, then deleting `onlyMinter` will not cause the test to fail. If the `onlyMinter` modifier doesn't block non-minters, then the unit tests won't catch it.

By the way,  as contrived as this example may seem, it is taken from a real codearena [report](https://code4rena.com/reports/2023-01-rabbithole#h-01-bad-implementation-in-minter-access-control-for-rabbitholereceipt-and-rabbitholetickets-contracts).

## Off by One Errors and Boundry Conditions

Mutation tests can be useful for catching off-by-one errors. Consider the following mutation:

```solidity
uint256 public LIMIT = 5;

// original
function mint(uint256 amount) external {
	require(amount < LIMIT, "exceeds limit");
}

// mutation
function mint(uint256 amount) external {
	require(amount <= LIMIT, "exceeds limit");
}
```

If our unit tests set the `amount` to be 3 and 8, the code will have a 100% branch coverage with respect to this test. However, the mutation tests will fail because the strict inequality was replaced with an inequality and the test still passed. This is because the tests do not accurately express the intended functionality. Specifically, the tests should enforce if the upper limit is 4 or 5. Testing values for `amount` like 3 or 8 do not fully define the smart contract specification for this function. 

## Vertigo-rs

[RareSkills](https://www.rareskills.io/) actively maintains a mutation testing tool for Solidity, [vertigo-rs](https://github.com/RareSkills/vertigo-rs). This was forked from the [vertigo](https://github.com/JoranHonig/vertigo) repo which is no longer maintained. Support for the Foundry framework has been added. The tool works with Foundry, Hardhat, and Truffle. Instructions to run the tool are in the Readme. No modifications to the Solidity codebase or tests are required. Simply close the repository, install the dependencies, then run it in the Solidity project that you are are testing.

## Other Mutation Testing Tools

Although vertigo-rs is the only tool that automatically runs the test suite, there are other noteable tools for generating mutations (but they don't support automatically re-running the test suite and summarizing the results).

- [Gambit](https://docs.certora.com/en/latest/docs/gambit/index.html) by Certora
- [Universal Mutator](https://github.com/sambacha/universalmutator/tree/new-solidity-rules) by [sambucha](https://github.com/sambacha)

There are other tools, but they apparently are no longer maintained.

## Mutation Score

Tools for languages besides Solidity sometimes provide a `mutation score`. This the the percentage of mutants that were killed. If 100% of the mutants were killed, then the unit tests can be relied upon to detect unwanted or accidental changes in the codebase.

For very large codebases, having a 100% score may be impractical. Solidity smart contracts are quite small compared to traditional codebases, such as most backend and frontend applications. Aiming for a 100% mutation score for codebases that large may be infeasible. But because Solidity smart contracts are relatively small, and bugs are catastrophic, surviving mutants should be scrutanized carefully.

## Limitations of Mutation Testing

Because mutation testing tests the quality of unit tests, and unit tests are generally stateless, mutation testing cannot naturally illuminate that stateful business logic is testing properly.

Mutation testing can create hundreds of mutations, but for the sake of time, most tools only run a subset of them. This means important mutations that uncover bugs in the test suite may be missed.

## Learn More

This material is part of our [Solidity bootcamp](https://www.rareskills.io/solidity-bootcamp). You can also learn Solidity for free with our free [Solidity course](https://www.rareskills.io/learn-solidity).

*Originally Published April 14, 2023*
