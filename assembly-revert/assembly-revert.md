# Assembly revert

Reverting transactions using inline assembly can be more gas-efficient than using the high-level Solidity `revert` or `require` statement. In this guide, we’ll explore how the different types of reverts in Solidity work under the hood by simulating their implementations in assembly.

The example below shows that the `revert` statement in the assembly version reduces the gas cost from 157 gas to 126 gas, saving 31 gas:

![A screenshot showing how Assembly `revert` is more gas-efficient compared to the regular Solidity `revert`](https://r2media.rareskills.io/assembly-revert-images/image.png)

As a pre-requisite, we assume that you have read the [Try Catch and All the Ways Solidity Reverts](https://www.rareskills.io/post/try-catch-solidity) article as well as our [article on ABI encoding](https://www.rareskills.io/post/abi-encoding).

In the EVM, memory is a long array of bytes that are byte-indexed. That is, we can read and write bytes based on their index. Even though memory is bytes indexed, we typically read and write 32 bytes at a time.

## `mstore` in assembly and how it works

Reverting with assembly depends heavily on the [Yul](https://docs.soliditylang.org/en/latest/yul.html) `mstore` opcode to store data in memory, so let’s deeply explore that opcode first.

The `mstore` opcode takes two arguments:

1. **Memory location**: The byte address where the data will be stored.
2. **Data**: The 32-byte data to be stored.

The following is an example of how to use `mstore`:

```solidity
assembly {
     mstore(memoryLocation, dataToStore)
}
```

If you want to store 32 bytes of `0xFF`at the memory location `0x00`, you would write:

```solidity
assembly {
    mstore(
        0x00, 
        0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    )
}
```

This stores the full 32-byte value starting at index `0x00`. If you instead want to store the value at memory location `0x01`, you would write:

```solidity

assembly {
    mstore(
        0x01, 
        0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    )
}

```

This shifts the start of the data by one byte, and the first byte at `0x00` will remain unaffected. The diagram below shows how `mstore` stores data in memory:

![A diagram showing how memory looks like after mstore is executed](https://r2media.rareskills.io/assembly-revert-images/image_1.png)

Note that even though we specified writing to byte index `0` in `mstore(0, ...)`, we wrote to 0 — and also the following 31 bytes — `mstore` writes 32 bytes at a time. 

### Implicit data padding in `mstore`

If we specify fewer than 32 bytes (64 hex chars) in the second argument, the Solidity compiler will left-pad it with zeros (the more significant bytes) until the value is 32 bytes long, then it will write those 32 bytes starting at the byte index specified in the first argument of `mstore`.

Consider the following example:

```solidity
assembly {
    mstore(0x00, 0xff)
}
```

The code above stores the data as `0x00000000000000000000000000000000000000000000000000000000000000ff` in memory, with `0xff` occupying the last byte and the rest of the preceding 31 bytes filled with zeros. 

In other words, `mstore(0x00, 0xff)` implicitly becomes `mstore(0x00, 0x00000000000000000000000000000000000000000000000000000000000000ff)`

The result of the value in memory is shown here:

![A diagram showing 0xff written to the 31st byte in EVM memory](https://r2media.rareskills.io/assembly-revert-images/image_2.png)

Recall that `mstore` writes 32 bytes, but in this case, 31 of those bytes are zeros, spanning from the `0th` byte to the `30th` byte inclusive. This means any data within the byte range 0-31 will be overwritten with zeros.

We can see how it looks like in memory if we return the stored data as shown in the screenshot below:

![A screenshot from Remix showing the revert values returned from a function](https://r2media.rareskills.io/assembly-revert-images/image_3.png)

The diagram below shows how `mstore` implicitly left-pads various hex values, with the first row being the example we just looked.

![A diagram showing how values smaller than 32 bytes are left-padded when stored to byte 0](https://r2media.rareskills.io/assembly-revert-images/mstore.jpg)

## Using `mstore8` to store data in memory

Alternatively, we can use the `mstore8` opcode, which is similar to `mstore` but stores just one byte of data at a specific memory location.

```solidity
assembly {
     mstore8(memoryLocation, exactlyOneByteOfData)
}
```

For example, if we we want to store a single byte of data (`0xff`) at the `31st` byte, we can directly store it using `mstore8` like this:

```solidity
assembly {
    mstore8(31, 0xff)
}
```

And the output will be the same as using `mstore`, with `0xff` occupying the last byte.

![return value of memory in bytes](https://r2media.rareskills.io/assembly-revert-images/Screenshot_2024-09-28_at_6.31.02_PM.png)

![mstore8 opcode diagram example](https://r2media.rareskills.io/assembly-revert-images/image_4.png)

The key difference between `mstore8` and `mstore` is that `mstore8` doesn’t add 31 extra zeros that would overwrite previously stored data spanning from the `0th` to the `31st` byte, unlike `mstore`.

### Writing `0xff` with zeros on the right using `mstore`

If you want to write `0xff` in the `0th` byte using `mstore` instead of `mstore8`, you can store `0xff` as the first byte and pad the remaining 31 bytes with zeros, as shown below:

```solidity
assembly {
    mstore(
        0x00, 0xff00000000000000000000000000000000000000000000000000000000000000
    )
}
```

This will store the value exactly as you specified, with `0xff` at the beginning and the remaining bytes as zeros:

![mstore opcode visual diagram](https://r2media.rareskills.io/assembly-revert-images/image_5.png)

Here is a test run of the code in Remix:

![solidity code using mstore](https://r2media.rareskills.io/assembly-revert-images/image_6.png)

That seems like a lot of zeros, right? Alternatively, we can use `mstore8` to store one byte of data at a specific memory location. In the example below, we used `mstore8` to store `0xff` at the `0th` byte:

![solidity code using mstore8](https://r2media.rareskills.io/assembly-revert-images/image_7.png)

This is a much more compact code that does the same thing as the one in the previous screenshot, except that it does not write zeros into bytes 1 to 31.

Note that:

- `mstore` stores a full 32 bytes of data, while `mstore8` only stores a single byte at the specified memory location.
- When using `mstore`, if your data is fewer than 32 bytes, it will automatically be padded with zero bytes on the left (the lower-indexed memory location) to fill the 32 bytes. These zeros will overwrite any other memory contents that were previously there.

## You can overwrite data in memory if you write to the same location multiple times

Remember when we mentioned that the extra zeros `mstore` left-pads will overwrite any existing content within the `0th` to `30th` byte range? Let’s explore how that overwrite happens.

If we write `0xCC` to the `0th` byte using `mstore8`:

```solidity
assembly {
   mstore8(0, 0xCC)
}
```

We will now have `0xCC` at the `0th` position, while the rest of the memory remains unchanged, as illustrated in the diagram below.

![mstore8 opcode usage example](https://r2media.rareskills.io/assembly-revert-images/image_8.png)

Subsequently, if we store `0xFF` using `mstore(0, 0xFF)` like so:

```solidity
assembly {
   mstore8(0, 0xCC)
   mstore(0, 0xFF)
}
```

`0xFF` will overwrite the previously stored `0xCC` at the `0th` byte and fill the entire 32-byte slot (from the `0th` to the `31st` byte) with `0xFF`. 

*Recall that `mstore` will write data to the entire 32-bytes slot and if we have fewer than 32 bytes, it will pad the remaining bytes with zeros like so:*

```solidity
0x00000000000000000000000000000000000000000000000000000000000000**FF**
```

The animation below shows how this overwrite happens:

<video src="https://r2media.rareskills.io/assembly-revert-images/Mstore8Scene.mp4" type="video/mp4" autoplay loop muted controls></video>

This demonstrates that the 31 padded zeros of `mstore` actually alters the contents of memory.

## How to remember the `mstore` padding

Instead of memorizing that `mstore` left-pads hex values with zero, we can consider that `mstore(0, 0xff)` is completely equivalent to `mstore(0, 255)`. 

In other words, `mstore(0, 255)` is saying “store the number 255 in the 32 bytes starting at byte 0, with byte 0 holding the most significant bytes.”

Since 255 is a "small number" compared to what a 32 byte number can hold (the [maximum value of uint256](https://www.rareskills.io/post/uint-max-value-solidity)), only the least significant bits will be used. The least significant bits are on the right, but the significant bits on the left are set to zero.

Similarly, the number `0xff00000000000000000000000000000000000000000000000000000000000000` is quite large.

In decimal, it is `115339776388732929035197660848497720713218148788040405586178452820382218977280`. Therefore, it uses up the most significant bits, which are on the left.

## Using the stored data to revert

We’ve seen how to store data in memory with `mstore`. During a revert, we need to store the error data in memory and return it as the revert error message. 

The `revert` opcode takes two arguments: the starting memory slot and the total size of data we intend to return.

```solidity
revert(startingMemorySlot, totalMemorySize)
```

From here onward we will show how to mimic the behavior of Solidity revert in the following situations:

1. Reverting without a reason string
2. Reverting with a custom error
3. Reverting with a custom error and reason string

## 1. Revert without a reason (message)

For a simple revert without a message, the assembly code `revert(0,0)` is equivalent to `revert()` in Solidity in behavior and gas cost. It does not return any data to the caller. 

Under the hood, using `revert(0,0)`, means “use no data” because the length of the data being referred to is zero. It is conventional to use memory location 0 as the starting point, but since we are returning nothing, we could do `revert(1,0)` and accomplish the same thing.

Here is a simple example of a revert without a reason using assembly:

```solidity

contract ContractA {
    function zero() external {
        assembly {
            revert(0,0) //<--- simple revert without reason
        }
    }
}
```

The screenshot below shows a [low-level call](https://rareskills.io/post/low-level-call) from `ContractA` to `ContractB` and how the low-level call returned `false` because `ContractB` reverted, and no data is returned since we are using `revert(0,0)`

![return value of a failed low-level call](https://r2media.rareskills.io/assembly-revert-images/image_9.png)

## 2a. Custom revert in assembly with no parameters

To illustrate how to simulate a custom error with no parameters using assembly, let’s use `revert Unauthorized()` as an example. 

We will store the custom revert’s [error function selector](https://www.rareskills.io/post/function-selector/) in a specific location in memory (`0x00` by convention) and revert will point to that location in memory.

Here is the Solidity code we will use as an example:

```solidity
contract CustomError {
    error Unauthorized();

    function revertCustomError() {
        revert Unauthorized();
    }
}
```

 We’ll follow the steps below to accomplish a custom revert in assembly:

1. Store the function selector in memory
2. Trigger the revert passing the selector’s memory location and the size of the selector (4 bytes) as arguments to `revert`

```solidity
assembly {
   mstore(memoryLocation, selector)
   revert(memoryLocation, sizeOfSelector)
}
```

### **1. Store the function selector**

When Solidity triggers a custom error, the return value is the ABI encoding of the custom error itself, which includes the [function selector](https://www.rareskills.io/post/function-selector) (the first four bytes of the keccak256 hash of the custom error signature).

Since we are using the custom error `Unauthorized()` as an example, we’ll first store the function selector (first four bytes of the keccak256 of `Unauthorized()` ) which will be `0x82b42900` padded with extra zeros to lengthen the value to 32 bytes to ensure that the actual four bytes of the function selector is written from byte 0 to byte 3 inclusive. Without this padding, the selector will not start at memory index 0.

```solidity
bytes32 selector = bytes32(abi.encodeWithSignature("Unauthorized()")); // 0x82b42900

assembly {    
    mstore(0x00, 0x82b4290000000000000000000000000000000000000000000000000000000000)
}
```

### **2. Triggering the revert**

We’ll now trigger the custom error with the revert statement below. Remember the template for the revert is `revert(startingMemorySlot, totalMemorySize)`.

```solidity
revert(0x00, 0x04)
```

The `0x00` is the memory location where we stored the error data while `0x04` (in hex) is the size of the error which is just 4 bytes. The entire revert code should now look like the code below:

```solidity
pragma solidity 0.8.27;

contract RevertErrorExample {
    function revertWithAssembly() public pure {
        assembly {
            mstore(
                0x00,
                0x82b4290000000000000000000000000000000000000000000000000000000000
            )
            revert(0x0, 0x04)
        }
    }
}
```

Here is the code we will use to compare our assembly implementation to the Solidity implementation:

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity 0.8.27;

contract RevertErrorExample {
    error Unauthorized();

        // assembly version
    function revertWithAssembly() public pure {
        assembly {
            mstore(
                0x00,
                0x82b4290000000000000000000000000000000000000000000000000000000000
            )

            revert(0x0, 0x04)
        }
    }

        // solidity version
    function revertWithoutAssembly() public pure {
        revert Unauthorized();
    }
}

```

The outcome is shown in the screenshot below. The only difference is in the gas cost. The screenshot shows that we saved 54 gas units by triggering revert via Assembly rather than in Solidity.

![revert with and without assembly gas usage difference](https://r2media.rareskills.io/assembly-revert-images/revert-update.drawio.jpg)

Also, in the code below, `callContractB` separately uses `try/catch` on `customRevertWithAssembly` and `customRevertWithoutAssembly` to parse the error, showing their behavior is the same.

![reverts in assembly and normal solidity showing the same error output](https://r2media.rareskills.io/assembly-revert-images/image_10.png)

### An alternative method to store the selector when a custom error has no parameters and trigger revert

When a custom error has no parameters, the function selector is the only relevant data to return. In that case, we can store the function selector without manually adding the extra zeros and revert it from the specific memory region we stored.

For example, instead of padding with zeros like this:

```solidity
assembly{
    mstore(
           0x00,
           0x82b4290000000000000000000000000000000000000000000000000000000000
    )
}
```

We can write it without manually padding it with zeros :

```solidity
assembly{
    mstore(0x00,0x82b42900)
}
```

In memory, the zeros will be added in the left of the function selector from the `0th` byte to the `27th` byte, while the actual selector will be stored from the `28th` byte to the `31st`. 

In other words `0x82b42900` gets expanded to `0000000000000000000000000000000000000000000000000000000082b42900` and stored in bytes `0` to `31` as shown below:

![using mstore to store the function selector in memory](https://r2media.rareskills.io/assembly-revert-images/image_11.png)

Since the function selector is now at the 28th byte (`0x1c` in hex), you can revert from this location instead of `0x00`, as shown below:

```solidity
assembly {
  mstore(0x00,0x82b42900)
  revert(0x1c, 0x04)
}
```

## 2b. Custom revert in assembly with parameters

If the custom error has parameters, we’ll need to ABI encode the arguments as well since it will be part of the revert return data. Assuming it has an address as an argument, we will store the argument in memory and point the revert arguments to both the selector and the address in memory.

As an example, let’s replicate the custom revert `Unauthorized(address)` in assembly.

```solidity
contract CustomError {
    error Unauthorized(address caller);

    function revertCustomError() {
        revert Unauthorized(msg.sender);
    }
}
```

The steps to replicate a custom revert  with arguments in assembly is similar to the one without arguments, the only difference is that we’ll need to store the arguments (in this case, the `address`) as part of the return data. We’ll follow the steps below:

1. Store the function selector in memory for the custom error
2. Store the argument in memory after the selector
3. Trigger the revert by passing the starting memory location and the total size (4 bytes for the selector + argument size) to the `revert` function

### 1. Store the function selector in memory for the custom error

Just like in the custom error without parameters, we’ll need to first derive the function selector like so:

```solidity
bytes4 selector = bytes4(abi.encodeWithSignature("Unauthorized(address)")
    ); 
```

And the selector will be `0x8e4a23d6`. We’ll now go ahead to store the selector starting at the `0x00` memory location with `mstore` as shown below:

```solidity
assembly{
    // Store the function selector at the memoryy location `0x00`
    mstore(0x00, 0x8e4a23d600000000000000000000000000000000000000000000000000000000)
}
```

### 2. Store the argument in memory after the selector

After writing the function selector to memory starting from the `0th` byte, we’ll now store the `address` from the `4th` byte as shown below:

```solidity
assembly{
    //...
    // Store the address
    // *Note that `caller()` in assembly is the same as `msg.sender` in Solidity.*
    mstore(0x04, caller())
}
```

 The function `caller()` will return the address upcasted to 32 bytes. So if the original address was `0x5B38Da6a701c568545dCfcB03FcB875f56beddC4` `caller()` will return `0x00000000000000000000000005B38Da6a701c568545dCfcB03FcB875f56beddC4` and this is the 32-byte value `mstore` will place in memory starting at byte 4.

### 3. Trigger the revert

And finally, we can now trigger the revert with the starting memory location and the total size the data we’ve stored so far (4 bytes for the selector + 32 bytes for the address) occupies 36 bytes (hex 0x24) as arguments as shown below:

```solidity
function customRevertWithAssembly() public pure {
    assembly {
        //...
        // 4 bytes for selector + 32 bytes for the address
        revert(0x00, 0x24)
    }
        
}
```

And this is how the entire code for the custom revert with parameters in assembly will look like:

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.27;

contract A { 
    function customRevertWithAssembly() public view {
        assembly {
            // Store the function selector at the memory location `0x00`
            mstore(0x00, 0x8e4a23d600000000000000000000000000000000000000000000000000000000)
                     
            // Store the address
            // N*ote that `caller()` in assembly is the same as `msg.sender` in Solidity.*
            mstore(0x04, caller())

            // 4 bytes for selector + 32 bytes for the address
            revert(0x00, 0x24)
        }
    }
}
```

This is how Solidity will store the revert data in memory and the result of the revert data will be returned eventually.

<video src="https://r2media.rareskills.io/assembly-revert-images/Memoryanim.mp4" type="video/mp4" autoplay loop muted controls></video>

## 3. Revert with a reason in assembly

When a revert with a reason string such as `revert("reason")` is triggered, the reverting contract returns the ABI encoding of `Error(string)`, along with the string argument. This is the same as how `require` with a reason works in Solidity.

To simulate the revert with a reason string in assembly, we need to ABI encode the same function and the string argument in memory.

Let’s use `revert(“Unauthorized”)` as an example:

```solidity
contract A {
    function revertWithAString() external pure {
        revert("Unauthorized");
    }
}
```

If we trigger the `revert("Unauthorized");` function in the contract above, the result will look like the example below.

![interpreting the raw revert error output](https://r2media.rareskills.io/assembly-revert-images/image_12.png)

In this section, we’ll replicate the revert with a string behavior in Solidity using assembly by following the steps below:

1. Store the function selector for `Error(string)` in memory
2. Store the offset to the error message string
3. Store the length of the error message string
4. Store the actual error message
5. Trigger the revert

Below is a quick representation of the above steps in assembly code:

```solidity
contract RevertErrorExample {

    function revertWithAssembly() public pure {
    
        assembly {
        
            // store the selector
            mstore(
                0x00,
                0x08c379a000000000000000000000000000000000000000000000000000000000
            ) 
            mstore(0x04, 0x20) // store the offset
            mstore(0x24, 0xc) // store the length of the string
            mstore(
                0x44,
                0x556E617574686F72697A65640000000000000000000000000000000000000000
            ) // store the actual data
            revert(0x00, 0x64) // trigger a revert
        }
        
    }
}
```

Let’s examine the assembly block line-by-line.

### 1. Store the function selector for `Error(string)`

We first store the [function selector](https://www.rareskills.io/post/function-selector/) at the starting memory location (`0x00`):

```solidity
assembly {
        mstore(0x00, 0x08c379a000000000000000000000000000000000000000000000000000000000) //Store the function selector
}
```

You can derive the function selector (the first 4 bytes of keccak256(`Error(string)`) with `abi.encodeWithSignature("Error(string)")` and then cast it to a 32-byte word with `bytes32` type like so:

```solidity
bytes32 selector = bytes32(abi.encodeWithSignature("Error(string)"));
```

The result will be:

```solidity
0x08c379a000000000000000000000000000000000000000000000000000000000
```

The first 4 bytes (`0x08c379a0`) is the selector padded with zeros to make up for the 32-bytes requirement.

![mstore the function selector in memory](https://r2media.rareskills.io/assembly-revert-images/image_13.png)

### 2. Store the offset to the error message string

The next part of the string error we store is the offset. The offset is 32 bytes (`0x20` in hex).

```solidity
mstore(0x04, 0x20) // 4 is 0x04 in hex
```

Remember, we mentioned that it's possible to overwrite memory if two memory locations overlap, right? Initially, the function selector was stored starting at the `0th` byte as a 32-byte word. Now, we are storing the offset starting at the 4th byte.

This means that the remaining data from the function selector (the padded zeros in this case) will be replaced, starting from the 4th byte, as shown in the diagram below:

![storing the offset to the error message string](https://r2media.rareskills.io/assembly-revert-images/image_14.png)

### 3. Store the length of the error message string

The third part of the string we need to store is the length of the string data. Recall that we stored the function selector at the `0x00` location, and it took up 4 bytes. Then the next memory location was the offset at memory location `0x04` which took up 32 bytes. 

That means 4 bytes selector + 32 bytes offset tells us that the next memory slot should be at 36 bytes, which is where we’ll store the length of the string.

The length of the string `Unauthorized` is 12 (`0xc`) bytes.

```solidity
mstore(0x24, 0xc) // 36 is 0x24 in hex
```

![location in memory to store the length of the error message string](https://r2media.rareskills.io/assembly-revert-images/image_15.png)

### 4. Store the actual error message string

The actual string `Unauthorized` is stored starting at 68 (`0x44`) bytes from the beginning which corresponds to the 4 bytes for the selector + 32 bytes offset + 32 bytes for the length. So far, we have written 100 bytes of data.

```solidity
mstore(0x44, "Unauthorized") //68 is 0x44 in hex
// We can store Unathorized as hex as well. Unauthorized in hex is ⤵️
// 0x556E617574686F72697A65640000000000000000000000000000000000000000
```

![example location in memory to store the raw error message string](https://r2media.rareskills.io/assembly-revert-images/image_16.png)

### 5. Trigger the revert:

The revert operation uses the starting memory location and the total size of the data to trigger the revert. 

The total size will be 100 (`0x64`) bytes by adding the 4 bytes for the selector, 32 bytes for the offset, 32 bytes for the length of the string, and 32 bytes for the string content "Unauthorized".

Remember the template for the revert in assembly:

```solidity
revert(StartingMemorySlot, totalMemorySize)
```

Here is how we’ll trigger the revert:

```solidity
revert(0x00, 0x64) // 100 is 0x64 in hex
```

So, the revert will return the following data with exactly 100 bytes when triggered:

![raw data of the crafted custom revert](https://r2media.rareskills.io/assembly-revert-images/image_17.png)

Even though the string `Unauthorized` does not use up the full 32 bytes, the receiver will know to only read 12 bytes of data due to the length parameter `0x0c`.

If we put all the steps together, we’ll arrive at this code:

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract ContractA {
    
    function revertWithAssembly() external pure {
        assembly {
            mstore(
                0x00,
                0x08c379a000000000000000000000000000000000000000000000000000000000
            ) // store the selector

            mstore(0x04, 0x20) // store the offset

            mstore(0x24, 0xc) // store the length of the string

            mstore(
                0x44,
                0x556E617574686F72697A65640000000000000000000000000000000000000000
            ) // store the actual data

            revert(0x00, 0x64) // trigger a revert
        }
    }
}
```

Here is a screenshot showing the output of the revert when you call the `revertWithAssembly()` function. The result is the same as what we saw when we triggered `revert(“Unauthorized”)` in Solidity.

![revert with assembly raw output](https://r2media.rareskills.io/assembly-revert-images/image_18.png)

However, the difference is in the amount of gas they both consume. Run the reverts in the following contracts to see the difference in gas cost. Below is the code we use to test the gas costs:

```solidity
// SPDX-License-Identifier: GPL-3.0

pragma solidity >=0.7.0 <0.9.0;

contract ContractA {
    function revertWithAssembly() external pure {
        assembly {
            mstore(
                0x00,
                0x08c379a000000000000000000000000000000000000000000000000000000000
            ) // store the selector
            mstore(0x04, 0x20) // store the offset
            mstore(0x24, 0xc) // store the length of the string
            mstore(
                0x44,
                0x556E617574686F72697A65640000000000000000000000000000000000000000
            ) // store the actual data
            revert(0x00, 0x64) // trigger a revert
        }
    }
}

contract ContractB {
    function revertWithoutAssembly() external pure {
        revert("Unauthorized");
    }
}
```

The illustration below shows the difference in gas cost for the `revertWithAssembly` function and the `revertWithoutAssembly`.

![revert with and without assembly cost difference](https://r2media.rareskills.io/assembly-revert-images/image_19.png)

From the above test, we saved `273` gas as the revert without assembly cost `428` gas while the revert with assembly cost `155` gas. The difference is `273`. 

To further verify that the error was properly formed, we can try to catch the error in a `try/catch` block as shown in the screenshot below:

![unauthorized error](https://r2media.rareskills.io/assembly-revert-images/image_20.png)

From the above screenshot, we can see that the error was caught in the `Error` `catch` block as expected and the reason was printed `Unauthorized`.

## Conclusion

In this guide, we have learned how revert works by implementing Solidity reverts manually using inline assembly.

We covered:

- how `mstore` and `mstore8` works
- how to mimic the following kind of reverts:
    - reverts without reasons
    - custom error reverts
    - and reverts with reason

We also saw how we could save some gas by using revert via assembly. I encourage you to experiment with it yourself, as that’s the best way to fully understand how it all comes together.

Happy coding

This article was written by [Eze Sunday](https://www.linkedin.com/in/ezesundayeze/) in collaboration with RareSkills.
