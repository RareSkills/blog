# Solidity test internal function

To test an internal solidity function, create a child contract that inherits from the contract being tested, wrap the parent contract’s internal function with an external one, then test the external function in the child.

Foundry calls this inheriting contract a "harness" though others call it a "fixture."

Don’t change the function to become virtual or public to make it easier to extend, you want to test the contract you will actually deploy.

Here is an example.

```solidity
contract InternalFunction {

	function calculateReward(uint256 depositTime) **internal** view returns (uint256 reward) {
		reward = (block.timestamp - depositTime) \* REWARD\_RATE\_PER\_SECOND;
	}
}
```

The function above gives a linear reward rate for each unit of time that passes by.

The fixture (or harness) would look like this:

```solidity
contract InternalFunctionHarness is InternalFunction {

	function calculateReward(uint256 depositTime) **external** view returns (uint256 reward) {
		reward = super.calculateReward(depositTime);
	}
}
```

When you call a parent function that has the same name as the child, you must use the super keyword of the function will call itself and go into infinite recursion.

Alternatively, you can explicitly label your test function as a harness or fixture as follows

```solidity
contract InternalFunctionHarness is InternalFunction {

	function calculateReward\_HARNESS(uint256 depositTime) **external** view returns (uint256 reward) {
		reward = calculateReward(depositTime);
	}
}
```

## Don’t change the function to be public

Changing the function to become public isn’t a good solution because this will increase the contract size. If a function doesn’t need to be public, then don’t make it public. It will increase the gas cost both for deployment, and the execution of the other functions.

When a [contract](https://www.rareskills.io/post/smart-contract-creation-cost) receives a transaction, it must compare the function selector to all the public ones in a linear or binary search. In either case, it has more selectors to search through. Furthermore, the added selector is added bytecode which increases the deployment cost.

## Don’t override virtual functions

Suppose we had the following contract:

```solidity
contract InternalFunction {

	function calculateReward(uint256 depositTime) **internal** view virtual returns (uint256 reward) {
		reward = (block.timestamp - depositTime) \* REWARD\_RATE\_PER\_SECOND;
	}
}
```

It could be tempting to simply override it on the the fixture for convenience, but this is not advisable since you end up duplicating code and if your implementation in the harness diverges from the parent contract, you won’t be actually testing your business logic anymore.

Note that this method forces us to copy and paste the original code:

```solidity
contract InternalFunctionHarness in InternalFunction {

	function calculateReward(uint256 depositTime) **external** view override returns (uint256 reward) {
		reward = (block.timestamp - depositTime) \* REWARD\_RATE\_PER\_SECOND;
	}
}
```

## What about testing private solidity functions?

There is no way to test private functions in solidity as they are not visible to the child contract. The distinction between an internal function and a private function doesn’t exist after the contract is compiled. Therefore, you can change private functions to be internal with no negative effect on the gas cost.

As an exercise for the reader, benchmark the following code to see that changing `foo` to be internal does not affect the gas cost.

```solidity
contract A {
    // change this to be private
    function foo() **internal** pure returns (uint256 f) {
        f = 2;
    }

    function bar() **internal** pure returns (uint256 b) {
        b = foo();
    }
}

contract B is A {
    // 146 gas: 0.8.7 no optimizer
    function baz() **external** pure returns (uint256 b) {
        b = bar();
    }
}
```

## Learn more

See our advanced [solidity bootcamp](https://www.rareskills.io/solidity-bootcamp) to learn more advanced testing methodologies.

We also have a free [solidity tutorial](https://www.rareskills.io/learn-solidity) to get you started.

*Originally Published April 6, 2023*