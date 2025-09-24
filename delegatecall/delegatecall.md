# Delegatecall: The Detailed and Animated Guide

This article explains how delegatecall works in detail. The **E**thereum **V**irtual **M**achine (**EVM**) offers four **opcodes** for making calls between contracts:

- **`CALL (F1)`**
- **`CALLCODE (F2)`**
- **`STATICCALL (FA)`**
- and **`DELEGATECALL (F4)`**.

Notably, the **`CALLCODE`** opcode has been deprecated since Solidity v5, being replaced by **`DELEGATECALL`**. These opcodes have a direct implementation in Solidity and can be executed as methods of variables of type `address`.

To gain a better understanding of how delegatecall works, let's first review the functionality of the **`CALL`** opcode.

## CALL

To demonstrate <span style="color: Red;">call</span> consider the following contract:

```solidity
contract Called {
    uint public number;

    function increment() public {
        number++;
    }
}
```

The most direct way to execute the <span style="color: Red;">increment()</span> function from another contract is by utilizing the <span style="color: Red;">Called</span> **contract interface**. In this recipe, we can execute the function with a statement as simple as <span style="color: Red;">called.increment()</span> where <span style="color: Red;">called</span> is the address of <span style="color: Red;">Called</span>. But calling <span style="color: Red;">increment()</span> can also be achieved using a low-level <span style="color: Red;">call</span> as show in the contract below:

```solidity
contract Caller {

    address constant public calledAddress = 0xd9145CCE52D386f254917e481eB44e9943F39138; // Called's address

    function callIncrement() public {
        calledAddress.call(abi.encodeWithSignature("increment()"));
    }
}
```

Every variable of type address, such as the <span style="color: Red;">calledAddress</span> variable, has a method named <span style="color: Red;">call</span>. This method expects as the parameter the input data to be executed in the transaction, that is, [ABI encoded](https://www.rareskills.io/post/abi-encoding) calldata. In the case mentioned above, the input data must correspond to the signature of the `increment()` function, with [function selector](https://www.rareskills.io/post/function-selector) `0xd09de08a`. We employ the abi.encodeWithSignature method to generate this signature from the function definition.

If you execute the `callIncrement` function in the `Caller` contract, you will observe that the state variable `number` in `Called` will increment by 1. The `call` method **does not verify whether the destination address actually corresponds to an existing contract, neither if it contains the specified function**.

The call transaction is visualized in the video below:

<video src="https://video.wixstatic.com/video/706568_c1abfd87d5584f2fa60f6fa4c1c49569/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

## Call returns a tuple

The `call` method returns a tuple with two values. The first value is a boolean indicating the success or failure of the transaction. The second value, of type bytes, holds the return value of the function executed by the `call`, ABI-encoded, if any.

To retrieve the `call` return, we can modify the `callIncrement` function as follows:

```solidity
function callIncrement() public {
    (bool success, bytes memory data) = called.call(
        abi.encodeWithSignature("increment()")
    );
}
```

The `call` method never reverts. If the transaction isn't `successful`, success will be false, and the programmer needs to handle this accordingly.

## Handling Call Failures

Let's modify the contract above to include another call to a non-existent function, as shown below.

```solidity
contract Caller {
    address public constant calledAddress = 0xd9145CCE52D386f254917e481eB44e9943F39138;

    function callIncrement() public {
        (bool success, bytes memory data) = called.call(
            abi.encodeWithSignature("increment()")
        );

        if (!success) {
            revert("Something went wrong");
        }
    }
        // calls a non-existent function
    function callWrong() public {
        (bool success, bytes memory data) = called.call(
            abi.encodeWithSignature("thisFunctionDoesNotExist()")
        );

        if (!success) {
            revert("Something went wrong");
        }
    }
}
```

I intentionally created two functions: one with the correct increment function signature and another with an invalid signature. The first function will return `true` for `success`, while the second will return `false`. The return boolean is explicitly handled, and the transaction will revert if `success` is false.

We must be careful to track if the call succeeded or not, and we'll revisit this issue shortly.

## What the EVM does under the hood

The purpose of the `increment` function is to increment the state variable called `number`. Since the EVM does not have knowledge of state variables but **operates on storage slots**, what the function actually does is to increase the value in the first slot of the storage, known as slot 0. This operation occurs within the storage of the `Called` contract.

![caller contract using the call function to a called contract](https://static.wixstatic.com/media/706568_ac35840fb9c349189d9504a644afcdd3~mv2.png/v1/fill/w_459,h_265,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_ac35840fb9c349189d9504a644afcdd3~mv2.png)

Having reviewed how to use the `call` method will help us form an idea about how to use `delegatecall`.

## DELEGATECALL

**A contract that makes a delegatecall to a target smart contract executes the logic of the target contract inside its own environment.**

One mental model is that it copies the code of the target smart contract and runs that code itself. The targeted smart contract is commonly referred to as the "implementation contract".

<video src="https://video.wixstatic.com/video/706568_26ef426f46e5484b96d39397f88d26b4/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

Just like `call`, `delegatecall` also has the input data to be executed by the target contract as a parameter.

Here is the code for the `Called` contract, corresponding to the animation above, which runs in the `Caller`'s environment:

```solidity
contract Called {
    uint public number;

    function increment() public {
        number++;
    }
}
```

And the code for `Caller`

```solidity
contract Caller {
    uint public number;

    function callIncrement(address _calledAddress) public {
        _calledAddress.delegatecall(
            abi.encodeWithSignature("increment()")
        );
    }
}
```

This **`delegatecall`** will execute the `increment` function; however, the execution will occur with a crucial difference. The `Caller` contract's storage will be modified, **NOT** the `Called` storage. It's as if the `Caller` contract borrowed `Called` code to execute in its own context.

The diagram below further illustrates how **`delegatecall`** modifies the `Caller`'s storage rather than the `Called`'s.

![caller contract using the call function to a called contract](https://static.wixstatic.com/media/706568_06f0e82e46c444c5815b73065d27410b~mv2.png/v1/fill/w_479,h_277,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_06f0e82e46c444c5815b73065d27410b~mv2.png)

The image below illustrates the distinction between executing the increment function using `call` and **`delegatecall`**.

![The distinction between executing the increment function using call and delegatecall.](https://static.wixstatic.com/media/706568_8aa3b0a1317141e2a9975dbbaeef6098~mv2.png/v1/fill/w_456,h_440,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_8aa3b0a1317141e2a9975dbbaeef6098~mv2.png)

## Storage slot collision

The contract issuing a **`delegatecall`** must be extremely careful to predict which of its storage slots will get modified. The previous example worked perfectly because `Caller` didn't use the state variable in slot 0. A common bug when using **`delegatecall`** is forgetting this fact. Let's see an example of this.

```solidity
contract Called {
    uint public number;

    function increment() public {
        number++;
    }
}

contract Caller {
        // there is a new storage variable here
    address public calledAddress = 0xd9145CCE52D386f254917e481eB44e9943F39138;

    uint public myNumber;

    function callIncrement() public {
        called.delegatecall(
            abi.encodeWithSignature("increment()")
        );
    }

}
```

Note that in the updated contract above, the content of slot **`0`** is the address of the `Called` contract, while the `myNumber` variable is now stored in slot **`1`**.

If you deploy the provided contracts and execute the `callIncrement` function, slot 0 of the `Caller` storage will be incremented, but the `calledAddress` variable is there, not the `myNumber` variable.

The following video illustrates this bug:

<video src="https://video.wixstatic.com/video/706568_4b1accda1bc44865822efef9df65dc2e/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

Let's illustrate what happened below.

![storage slot of the caller when using delegatecall()](https://static.wixstatic.com/media/706568_474fe99034d7457eb71369385b164221~mv2.png/v1/fill/w_444,h_434,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_474fe99034d7457eb71369385b164221~mv2.png)

Therefore, caution must be exercised when using **`delegatecall`**, as it can inadvertently break our contract. In the example above, it is likely not the programmer's intention to alter the `calledAddress` variable through the `callIncrement` function.

Let's make a small change to `Caller` by moving the state variable `myNumber` to slot 0.

```solidity
contract Caller {

    uint public myNumber;

    address public calledAddress = 0xd9145CCE52D386f254917e481eB44e9943F39138;

    function callIncrement() public {
        called.delegatecall(
            abi.encodeWithSignature("increment()")
        );
    }
}
```

Now, when executing the `callIncrement` function, the `myNumber` variable will be incremented, as this is the purpose of the `increment` function. I purposely chose the name of the variable in `Caller` differently from that in `Called` to demonstrate that **the name of the variable doesn't matter; what is fundamental is which slot it is in**. Aligning the state variables of both contracts is crucial for the proper functioning of delegatecall.

## Decouple implementation from data

One of the most important uses of **`delegatecall`** is to decouple the contract where the data is stored, such as `Caller` in this case, from the contract where the execution logic resides, such as `Called`. Therefore, if one wishes to alter the execution logic, one can simply replace `Called` with another contract and update the reference to the implementation contract, without touching the storage. `Caller` is no longer constrained by the functions it has, it can delegatecall the functions it needs from other contracts.

In case there is a need to change the execution logic, for example, subtracting the value of `myNumber` by 1 unit instead of adding it, you can create a new implementation contract, as shown below.

```solidity
contract NewCalled {

    uint public number;

    function increment() public {
        number = number - 1;
    }
}
```

Unfortunately, it's not possible to change the name of the function that will be called, as doing so would alter its signature.

After creating the new implementation contract, `NewCalled`, one can simply deploy this new contract and change the `calledAddress` state variable in `Caller`. Of course, `Caller` would need to have a mechanism to change the address it is issuing the `delegateCall` to, which we did not include to keep the code concise.

We have successfully modified the business logic utilized by the Caller contract. **Separating data from execution logic allows us to create upgradable smart contracts in Solidity.**

![delegatecall() allows a contract to separate data and business logic](https://static.wixstatic.com/media/706568_41c80f67f6a2487e8094cbeb661ccd27~mv2.png/v1/fill/w_484,h_339,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_41c80f67f6a2487e8094cbeb661ccd27~mv2.png)

In the image above, the contract on the left handles both the data and the logic. On the right, the top contract holds the data, but the mechanism to update the data is held in the logic contract. To update the data, a delegatecall is made to the logic contract.

## Handling delegatecall returns

Just like **`call`**, **`delegatecall`** also returns a tuple containing two values: a boolean indicating the success of the execution and the return of the function executed via **`delegatecall`**, in bytes. To see how to handle this return, let's write a new example.

```solidity
contract Called {
    function calculateDiscountPrice(
        uint256 amount,
        uint256 discountRate
    ) public pure returns (uint) {
        return amount - (amount * _discountRate)/100;
    }
}

contract Caller {
    uint public price = 200;
    uint public discountRate = 10;
    address public called;

    function setCalled(address _called) public {
        called = _called;
    }

    function setDiscountPrice() public  {
        (bool success, bytes memory data) = called.delegatecall(
            abi.encodeWithSignature(
                "calculateDiscountPrice(uint256,uint256)",
                price,
            discountRate)
        );

        if (success) {
            uint newPrice = abi.decode(data, (uint256));
            price = newPrice;
        }
    }
}
```

The `Called` contract contains logic to calculate a discount price. We utilize this logic by executing the `calculateDiscountPrice` function via **`delegatecall`**. This function returns a value, which we must decode using **`abi.decode`**. Before making any decisions based on this return value, it's crucial to check whether the function was executed successfully, otherwise we might try to parse a return that isn't there or end up parsing a revert reason string.

## When call and delegatecall return false

A crucial point to understand is when the success value will be `true` or `false`. **Essentially, it depends on whether the function being executed will revert or not.** There are three ways an execution can revert:

- if it encounters a REVERT opcode,
- if it runs out of gas,
- if it attempts something prohibited, such as dividing by zero.

If the function being executed via `delegatecall` (or **`call`**) encounters any of these conditions, it will revert, and the return value of the `delegatecall` will be false

A question that often confuses developers is why a `delegatecall` for a non-existent contract doesn't revert and still reports that the execution was successful. Based on what we said, an empty address will never meet one of the three conditions for reverting, so it will never revert.

## Another example of storage variable gotchas

Let's make slight modification to the code above to give another example of bugs related to the storage layout.

The `Caller` contract still invokes an implementation contract via **`delegatecall`**, but now the `Called` contract reads a value from a state variable. This might seem like a minor alteration, but it actually leads to disaster. Can you figure out why?

```solidity
contract Called {
    uint public discountRate = 20;

    function calculateDiscountPrice(uint256 amount) public pure returns (uint) {
        return amount - (amount * discountRate)/100;
    }
}

contract Caller {
    uint public price = 200;
    address public called;
    function setCalled(address _called) public {
        called = _called;
    }
    function setDiscount() public  {
        (bool success, bytes memory data) =called.delegatecall(
            abi.encodeWithSignature(
                "calculateDiscountPrice(uint256)",
                price
            )
        );

        if (success) {
            uint newPrice = abi.decode(data, (uint256));
            price = newPrice;
        }
    }
}
```

The problem arises because `calculateDiscountPrice` is reading a state variable, specifically the one in slot 0. Remember that in **`delegatecall`**, functions are executed in the storage of the calling contract. In other words, you may think you are using the `discountRate` variable from the `Called` contract to calculate the new `price`, but you are actually using the price variable from the `Caller` contract! The storage variables `Called.discountRate` and `Called.price` occupy slot 0.

You will receive a 200% discount, which is quite substantial (and will cause the function to revert, since the new calculated price will become negative, which is not allowed for a **uint** type variable).

## Immutable and Constant Variables in delegatecall: A Bug Story

Another tricky issue with **`delegatecall`** arises when immutable or constant variables are involved. Let's examine an example that many experienced Solidity programmers misunderstand:

```solidity
contract Caller {
    uint256 private immutable a = 3;
    function getValueDelegate(address called) public pure returns (uint256) {
        (bool success, bytes memory data) = called.delegatecall(
            abi.encodeWithSignature("getValue()"));
        return abi.decode(data, (uint256)); // is this 3 or 2?
    }
}

contract Called {
    uint256 private immutable a = 2;

    function getValue() public pure returns (uint256) {
        return a;
    }
}
```

The question is: when executing `getValueDelegate`, will the return be 2 or 3? Let's reason about it.

- The `getValueDelegate` function executes the `getValue` function, which supposedly returns the value corresponding to the state variable in slot 0.
- Since it's delegatecall, we should examine the slot in the calling contract, not the called contract.
- The value of variable `a` in `Caller` is 3, so the response must be 3. Nailed it.

Surprisingly, the correct answer is 2. WHY?!

**Immutable or constant state variables are not true state variables: they do not occupy a slot**. When we declare immutable variables, their value is hardcoded in the contract bytecode which is executed during the delegatecall. Thus, the `getValue` function returns the hardcoded value 2.

## msg.sender, msg.value and address(this)

If we use **`msg.sender`**, **`msg.value`** and **`address(this)`** within the `Called` contract, all these values will correspond to the `Caller` contract's **`msg.sender`**, **`msg.value`** and **`address(this)`** values. Let's remember of how delegatecall operates: everything occurs within the context of the caller contract. **The implementation contract merely provides the bytecode to be executed, nothing more.**

![delegatecall() lends the bytecode of the called contract to the caller contract](https://static.wixstatic.com/media/706568_348197999164437da8b94ea24390d50d~mv2.png/v1/fill/w_491,h_283,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_348197999164437da8b94ea24390d50d~mv2.png)

Let's apply this concept in an example. Let's consider the following code:

```solidity
contract Called {
    function getInfo() public payable returns (address, uint, address) {
        return (msg.sender, msg.value, address(this));
    }
}


contract Caller {
    function getDelegatedInfo(
        address _called
    ) public payable returns (address, uint, address) {
        (bool success, bytes memory data) = _called.delegatecall(
            abi.encodeWithSignature("getInfo()")
        );
        return abi.decode(data, (address, uint, address));
    }
}
```

In the `Called` contract, I'm utilizing **`msg.sender`**, **`msg.value`** and **`address(this)`**, and returning these values in the `getInfo` function. In the figure below, an execution of `getDelegateInfo` using **Remix** is depicted, displaying the returned values.

- **`msg.sender`** corresponds to the account that executed the transaction, specifically the first Remix default account, which is `0x5B38Da6a701c568545dCfcB03FcB875f56beddC4`.
- **`msg.value`** reflects the value of 1 ether, which was sent in the original transaction.
- **`address(this)`** is the address of the Caller contract, as one can see on the left side of the figure, rather than the address of the Called contract.

![delegatecall() lends the bytecode of the called contract to the caller contract](https://static.wixstatic.com/media/706568_79fd5b31a80b4416ae779fec7fb28425~mv2.png/v1/fill/w_493,h_285,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_79fd5b31a80b4416ae779fec7fb28425~mv2.png)

In Remix, we display log values for msg.sender (0), msg.value (1) and address(this) (2).

## msg.data and input data in delegatecall

The `msg.data` property returns the calldata of the context being executed. When `msg.data` is called in a function that is being executed directly via a transaction by an EOA, `msg.data` represents the input data of the transaction.

When we execute a call or a delegatecall, we specify as the argument the input data that will be executed in the implementation contract. So the original calldata differs from the calldata within the sub-context created by **`delegatecall`**, and consequently `msg.data` will differ.

![Image showing that msg.data with delegatecall() is different for the caller and called contracts](https://static.wixstatic.com/media/706568_3c4a8cca1d694925b183c972e00cca28~mv2.png/v1/fill/w_740,h_404,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_3c4a8cca1d694925b183c972e00cca28~mv2.png)

The code below will be used to demonstrate this.

```solidity
contract Called {
    function returnMsgData() public pure returns (bytes memory) {
        return msg.data;
    }
}

contract Caller {
    function delegateMsgData(
        address _called
    ) public returns (bytes memory data) {
        (, data) = _called.delegatecall(
            abi.encodeWithSignature("returnMsgData()"));
    }
}
```

The original transaction executes the `delegateMsgData` function, which requires a parameter of type address. As a result, the input data will consist of the function signature along with an address, ABI-encoded.

The `delegateMsgData` function, in turn, delegatecalls the `returnMsgData` function. To accomplish this, the calldata passed to the runtime must contain the signature of `returnMsgData`. Consequently, the value of `msg.data` inside `returnMsgData` is its own signature, given by `0x0b1c837f`.

In the image below, we can observe that the return of `returnMsgData` is its own signature, ABI-encoded.

![image of the return of returnMsgData is its own signature, ABI-encoded.](https://static.wixstatic.com/media/706568_b082c3003d734b67bf7a8118c2ec0a80~mv2.jpg/v1/fill/w_740,h_165,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/706568_b082c3003d734b67bf7a8118c2ec0a80~mv2.jpg)

The decoded output is the signature of the returnMsgData function, ABI-encoded as bytes.

## Codesize as a counterexample

We mentioned that we can conceive of delegatecall with the idea that we are borrowing the bytecode from the implementation contract and executing it in the calling contract. There is one exception to this, the `CODESIZE` opcode.

Suppose a smart contract has `CODESIZE` in its bytecode, `CODESIZE` returns the size of *that* contract. Codesize does not return the size of the caller's code during a delegatecall — it returns the size of code that was *delegatecalled*.

To demonstrate this property, we have provided the code below. In Solidity, `CODESIZE` can be executed in assembly via the `codesize()` function. We have two implementation contracts, `CalledA` and `CalledB`, which differ only by a local variable (`unused` in `ContractB` — that variable is absent in `ContractA`), intended to ensure that the contracts have different sizes. These contracts are called via **`delegatecall`** by the Caller contract's `getSizes` function.

```solidity
// codesize 1103
contract Caller {
    function getSizes(
        address _calledA,
        address _calledB
    ) public returns (uint sizeA, uint sizeB) {
        (, bytes memory dataA) = _calledA.delegatecall(
            abi.encodeWithSignature("getCodeSize()")
        );
        (, bytes memory dataB) = _calledB.delegatecall(
            abi.encodeWithSignature("getCodeSize()")
        );
        sizeA = abi.decode(dataA, (uint256));
        sizeB = abi.decode(dataB, (uint256));
    }
}

// codesize 174
contract CalledA {
    function getCodeSize() public pure returns (uint size) {
        assembly {
            size := codesize()
        }
    }
}

// codesize 180
contract CalledB {
    function getCodeSize() public pure returns (uint size) {
        uint unused = 100;
        assembly {
            size := codesize()
        }
    }
}

// You can use this contract to check the size of contracts
contract MeasureContractSize {
    function measureConctract(address c) external view returns (uint256 size){
        size = c.code.length;
    }
}
```

If the `codesize` function returned the size of the `Caller` contract, the values returned from `getSizes()` by delegate calling `ContractA` and `ContractB` would be the same. That is, they would be the size of `Caller`, which is 1103. However, as we can see in the figure below, the values are different, explicitly indicating that these are the sizes of `CalledA` and `CalledB`.

![Code size output CalledA and CalledB contract](https://static.wixstatic.com/media/706568_7363419278d24f6f9906150d34e7e5f7~mv2.png/v1/fill/w_740,h_185,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_7363419278d24f6f9906150d34e7e5f7~mv2.png)

### Delegatecall a delegatecall

One might wonder: What happens if a contract issues a **`delegatecall`** to a second contract that issues a **`delegatecall`** to a third contract? In such a case, the context will persist as that of the contract that initiated the first **`delegatecall`**, rather than the intermediate contract.

It works as follows:

- The `Caller` contract delegatecalls the function `logSender()` in the `CalledFirst` contract.
- This function is intended to emit an event that logs **`msg.sender`**.
- Additionally, the `CalledFirst` contract, apart from creating this log, also delegatecalls the `CalledLast` contract.
- The `CalledLast` contract also emits an event, which also logs the **`msg.sender`**.


A diagram depicting this flow is presented below.

![three contracts calling one another sequentially using delegatecall() and logsender()](https://static.wixstatic.com/media/706568_94ee7bccaabc4e2eab821defcde4d500~mv2.png/v1/fill/w_444,h_317,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_94ee7bccaabc4e2eab821defcde4d500~mv2.png)

Remember, all delegatecall does is borrow the bytecode of the delegatecalled contract. One way to visualize this is that the bytecode is temporarily "absorbed" into the calling contract. When we look at it that way, we see msg.sender is always the original msg.sender as everything is happening inside Caller. See the animation below:

<video src="https://video.wixstatic.com/video/706568_c3578d63394d4ddab30f8d64edfc4d90/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

Below we provide some source code to test the concept of a delegatecall to a delegatecall:

```solidity
contract Caller {
    address calledFirst = 0xF27374C91BF602603AC5C9DaCC19BE431E3501cb;
    function delegateCallToFirst() public {
        calledFirst.delegatecall(
            abi.encodeWithSignature("logSender()")
        );
    }
}

contract CalledFirst {
    event SenderAtCalledFirst(address sender);
    address constant calledLast = 0x1d142a62E2e98474093545D4A3A0f7DB9503B8BD;

    function logSender() public {
        emit SenderAtCalledFirst(msg.sender);
        calledLast.delegatecall(
            abi.encodeWithSignature("logSender()")
        );
    }
}

contract CalledLast {
    event SenderAtCalledLast(address sender);
    function logSender() public {
        emit SenderAtCalledLast(msg.sender);
    }
}
```

We might be led to think that the **`msg.sender`** in `CalledLast` will be the address of `CalledFirst`, since it was the one that called `CalledLast`, but this would not respect our model that the bytecode of the contract called via `delegatecall` is just borrowed, and that the context is always from the contract that executed the `delegatecall`.

The end result is that both **`msg.sender`** values correspond to the account that initiated the transaction with `Caller.delegateCallToFirst()`. This can be observed in the figure below, where we execute this process in Remix and capture the logs.

![result of msg.sender for both CalledFirst and CalledLas](https://static.wixstatic.com/media/706568_4aaead87a55c4fd6ae399b32e7abf5bc~mv2.png/v1/fill/w_740,h_363,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_4aaead87a55c4fd6ae399b32e7abf5bc~mv2.png)

msg.sender is the same in CalledFirst and CalledLast

One source of confusion is some might describe this operation as "`Caller` delegatecalls `CalledFirst` and `CalledFirst` delegatecalls `CalledLast`." But this makes it sound like `CalledFirst` is doing the delegatecall — that is not the case. `CalledFirst` is providing the bytecode to `Called` — and that bytecode is making a delegatecall to `CalledLast` — from `Called`.

Call from a delegatecall
Let's introduce a plot twist and modify the CalledFirst contract. Now, CalledFirst will invoke CalledLast using **`call`**, not **`delegatecall`**.

![three contracts calling one another sequentially using delegatecall() and logsender()](https://static.wixstatic.com/media/706568_4378e2e1267f4b459d0ab1416b5af2d7~mv2.png/v1/fill/w_462,h_330,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_4378e2e1267f4b459d0ab1416b5af2d7~mv2.png)

In other words, the CalledFirst contract needs to be updated to the following code:

```solidity
contract CalledFirst {
    event SenderAtCalledFirst(address sender);
    address constant calledLast = ...;

    function logSender() public {
        emit SenderAtCalledFirst(msg.sender);
        calledLast.call(
            abi.encodeWithSignature("logSender()")
        ); // this is new
    }
}
```

The question arises: What will be the **`msg.sender`** logged in the `SenderAtCalledLast` event? The following animation illustrates what happens:

<video src="https://video.wixstatic.com/video/706568_e72fbb2695704770bccd46f118b72ba5/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

![the msg.sender output when a contract is being called by another contract using delegatecall().](https://static.wixstatic.com/media/706568_290a68fb64d547909f8326251868e002~mv2.png/v1/fill/w_511,h_207,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_290a68fb64d547909f8326251868e002~mv2.png)

When `Caller` calls a function in `CalledFirst` via **`delegatecall`**, that function is executed within the context of `Caller`. Remember, `CalledFirst` merely "lends" its bytecode to be executed by `Caller`. At this point, it's as if we executed **`msg.sender`** in the `Caller` contract, which means that msg.sender is the address that initiated the transaction.

![the msg.sender when comparing to call functions: delegatecall and call](https://static.wixstatic.com/media/706568_41fbd53ba0bd49b18197cf251502cfb9~mv2.png/v1/fill/w_475,h_333,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_41fbd53ba0bd49b18197cf251502cfb9~mv2.png)

Now, `CalledFirst` calls `CalledLast`, but `CalledFirst` is being used in the context of `Caller`, so it's as if `Caller` made a call to `CalledLast`. In this case, **`msg.sender`** in `CalledLast` will be the address of `Caller`.

In the figure below, we observe the logs in Remix. Notice that this time the **`msg.sender`** values are different.

![Image of the msg.sender values of a contract calling another contract using delegatecall and call](https://static.wixstatic.com/media/706568_5aff230e5369466fab6205e304bbddfc~mv2.png/v1/fill/w_740,h_396,al_c,lg_1,q_85,enc_auto/706568_5aff230e5369466fab6205e304bbddfc~mv2.png)

msg.sender in CalledLast is the Caller address

**Exercise:** If Caller *calls* CalledFirst and CalledFirst *delegatecalls* CalledLast, and each contract logs msg.sender, which message sender will each contract log?

## Low-level delegatecall

In this section, we'll utilize **`delegatecall`** in **YUL** to explore its functionality at a deeper level. Functions in **YUL** closely resemble opcode syntax, making it beneficial to examine the definition of the **`DELEGATECALL`** opcode first.

**`DELEGATECALL`** takes 6 arguments from the stack, in order: **gas**, **address**, **argsOffset**, **argsSize**, **retOffset** and **retSize**, and returns a value to stack indicating whether the operation was carried out successfully (1) or not (0).

The explanation of each argument is as follows (taken from [evm.codes](http://evm.codes/)):

1. **gas**: amount of gas to send to the sub context to execute. The gas that is not used by the sub context is returned to this one.
2. **address**: the account which code to execute.
3. **argsOffset**: byte offset in the memory in bytes, the calldata of the sub context.
4. **argsSize**: byte size to copy (size of the calldata).
5. **retOffset**: byte offset in the memory in bytes, where to store the return data of the sub context.
6. **retSize**: byte size to copy (size of the return data).

Sending ether to a contract using delegatecall is not allowed (imagine the potential exploits if it were!). The `CALL` opcode, on the other hand, permits ether transfer and includes an additional parameter to indicate how much ether should be sent.

In YUL, the **`delegatecall`** function mirrors the **`DELEGATECALL`** opcode and includes the same 6 arguments mentioned above. Its syntax is:

```solidity
delegatecall(g, a, in, insize, out, outsize).
```

Below, we present a contract with two functions that perform the same action, executing a **`delegatecall`**. One is written in pure Solidity, and the other incorporates YUL.

```solidity
contract DelegateYUL {

    function delegateInSolidity(
        address _address
    ) public returns (bytes memory data) {
        (, data) = _address.delegatecall(
            abi.encodeWithSignature("sayOne()")
        );
    }

    function delegateInYUL(
        address _address
    ) public returns (uint data) {
        assembly {
            mstore(0x00, 0x34ee2172) // Load the calldata I intend to send into memory at 0x00. The first slot will become 0x0000000000000000000000000000000000000000000000000000000034ee2172
            let result := delegatecall(gas(), _address, 0x1c, 4, 0, 0x20) // The third parameter indicates the starting position in memory where the calldata is located, the fourth parameter specifies its size in bytes, and the fifth parameter specifies where the returned calldata, if any, should be stored in memory
            data := mload(0) // Read delegatecall return from memory
        }
    }
}

contract Called {
    function sayOne() public pure returns (uint) {
        return 1;
    }
}
```

In the `delegateInSolidity` function, I utilize the **`delegatecall`** method in Solidity, passing as a parameter the signature of the `sayOne` function, calculated using the `abi.encodeWithSignature` method.

If we don't know the size of the return in advance, don't worry, we can use the returndatacopy function later to handle this. In another article, when we delve deeper into writing upgradable contracts using delegatecall, we will cover all these details.

## EIP 150 and gas forwarded

A note on an issue regarding forwarded gas: We utilize the `gas()` function as the first parameter of **`delegatecall`**, which returns the available gas. This should indicate that we intend to forward all available gas. However, since the **Tangerine Whistle fork**, there has been a [cap of 63/64 of the total possible gas](https://www.rareskills.io/post/eip-150-and-the-63-64-rule-for-gas) for forwarding via **`delegatecall`** (and other opcodes). In other words, although the `gas()` function returns all available gas, only 63/64 of it is forwarded to the new subcontext, while 1/64 is retained.

## Conclusion

To conclude this article, let's summarize what we've learned. `Delegatecall` allows for the execution of functions defined in other contracts within the context of the calling contract. The called contract, also known as the implementation contract, merely provides its bytecode, and nothing within it is changed or fetched from its storage.

`Delegatecall` is employed to separate the contract where the data is stored from the contract where the business logic or function implementation is housed. **This forms the foundation of the most used pattern of contract upgradability in Solidity.** However, as we have observed, `delegatecall` must be utilized with great care, as unintentional changes to state variables can occur, potentially rendering the calling contract unusable.

## Learn More with RareSkills

For those new to Solidity, see our free [Solidity course](https://www.rareskills.io/learn-solidity). Intermediate Solidity developers please see our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp).

## Authorship

This article was written by [João Paulo Morais](https://www.linkedin.com/in/jpmorais/) in collaboration with RareSkills.

*Originally Published May 3*
