# Storage Slots in Solidity: Storage Allocation and Low-level assembly storage operations

This article examines the storage architecture of the Ethereum Smart Contracts. It explains how variables are kept in the EVM storage and how to read and write to storage slots using low-level assembly (Yul).

This information is a prerequisite to understanding how proxies in Solidity work and how to gas optimize smart contracts.

**Authorship**
This article was co-authored by Aymeric Taylor ([LinkedIn](https://www.linkedin.com/in/aymeric-russel-taylor/), [Twitter](https://twitter.com/AymerETH)), a research intern at RareSkills.

## Smart Contract Storage Architecture

Variables in a smart contract store their value in two primary locations: **storage** and **bytecode**.

![Variables store their value in either the bytecode or storage](https://static.wixstatic.com/media/706568_6721becb89494915adb6c9f4d1e9a83f~mv2.png/v1/fill/w_740,h_503,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_6721becb89494915adb6c9f4d1e9a83f~mv2.png)

### Bytecode

The **bytecode** stores immutable information. These include the values of `immutable` and `constant` variable types,

```solidity
contract ImmutableVariables{
    uint256 constant   myConstant = 100;
    uint256 immutable  myImmutable;
}
```

as well as the compiled source code (The source code is the **entire** text below).

```solidity
contract ImmutableVariables {
    uint256 constant myConstant = 100;
    uint256 immutable myImmutable;

    constructor(uint256 _myImmutable) {
        myImmutable = _myImmutable;
    }

    function doubleX() public pure returns (uint256) {
        uint256 x = 20;
        return x * 2;
    }
}
```

In the `doubleX()` function above, the value of hardcoded local variable such as `uint256 x = 20` will also be stored in the bytecode.

As this article focuses on covering the storage aspect, we will not discuss the bytecode in detail.

### Storage

The **storage** holds mutable information. Variables that store their value in the storage are called *state variables* or *storage variables*.

![storage variables store their data in the storage](https://static.wixstatic.com/media/706568_50c8b2280bf44d8b9bfe64e89b207a52~mv2.png/v1/fill/w_740,h_189,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_50c8b2280bf44d8b9bfe64e89b207a52~mv2.png)

Their value persists in the storage indefinitely, until further transactions alter them or the contract self-destructs.

Storage variables are variables of all types that are declared within the global scope of a contract (except for immutable and constant variables).

```solidity!
contract StorageVariables{
    uint256 x;
    address owner;
    mapping(address => uint256) balance;
    // and more...
}
```

When we interact with a storage variable, under the hood, we are actually reading and writing from the storage, specifically at the **storage slot** where the variable keeps its value.

## Storage slots

A smart contract's **storage** is organized into **storage slots**. Each slot has a fixed storage capacity of 256 bits or 32 bytes ($256 \div 8 = 32$).

![Storage slots visualized diagrammatically](https://static.wixstatic.com/media/706568_36b026974ae94ce8a3f61b3fbf315b05~mv2.png/v1/fill/w_740,h_326,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_36b026974ae94ce8a3f61b3fbf315b05~mv2.png)

**Storage slots** are indexed from $0$ to $2^{256} - 1$. These numbers act as a unique identifier for locating individual slots.

The Solidity compiler allocates storage space to storage variables in a sequential and deterministic manner, based on their declaration order within the contract.

Consider the contract below, it contains two storage variables: `uint256 x` and `uint256 y`.

```solidity
contract StorageVariables {
    uint256 public x; // first declared storage variable
    uint256 public y; // second declared storage variable
}
```

Since `x` is declared **first** and `y` is declared **second**, `x` is allocated to the first storage slot, **slot 0**, and `y` allocated to the second storage slot, **slot 1**. Thus, `x` will retain its value at slot 0, and `y` at slot 1.

![Animation of storage variables storing their value in their allocated storage slots](https://static.wixstatic.com/media/706568_ac190e8749df4363ac29f417c6a00efc~mv2.gif)

When queried, `x` and `y` will consistently read from the values stored in their respective storage slots. A variable cannot change its storage slot once the contract is deployed to the blockchain.

If the value of `x` and `y` is **not** initialized, it defaults to zero. **All storage variables default to zero until they are explicitly set.**

```solidity
contract StorageVariables {
    uint256 public x; // Uninitialized storage variable

    function return_uninitialized_X() public view returns (uint256) {
        return x; // returns zero
    }
}
```

To set the value of `x` to `20`, we can call the function `set_x(20)`.

```solidity
function set_x(uint256 value) external {
    x = value;
}
```

This transaction triggers a state change in slot 0, updating its state from <span style="color:Green;">0</span> to <span style="color:Red;">20</span>.

![State change animation of the variable x triggered by a function](https://static.wixstatic.com/media/706568_4c88284b4d7a49c5b787eef1bfd29a43~mv2.gif)

Essentially, all state changes made to a smart contract correspond to changes within these storage slots.

### Inside storage slots: 256-bit data

Individual storage slots store data in 256-bit format; It stores the bit representation of a storage variable's value.

In our previous example, `uint256 x` stores its value at **slot 0**. A `uint256` variable is 256 bit/32 bytes in size, therefore it will use up the 256 bit of storage space within slot 0 to store its value.

- **Before** calling `set_x(20)`, **slot 0** was at its default state (all zeros)

![storage slot default state visualized in text and raw bitsan](https://static.wixstatic.com/media/706568_ac6042ef20d7427da253812b36185b97~mv2.gif)

All the <span style="color:Green">green zeros</span> seen in the image above correspond to the bits that are used to store `x`'s value.

- **After** calling `set_x(20)`, slot 0's state was changed to the bit representation of **uint256 20**.

![Text and raw bit representation of Storage slot 0 keeping the value of 20](https://static.wixstatic.com/media/706568_c4e5c17749894730a6618499a981a8d3~mv2.gif)

Reading the contents of a storage slot in raw 256 bit format is less human readable, therefore, Solidity devs usually read it in hexadecimal format.

**Raw 256 bit:** `00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000`

**Hexadecimal format:**
`0x0000000000000000000000000000000000000000000000000000000000000014`

The **256** bit of ones and zeros can be reduced to just **64** hexadecimal numbers. 1 hexadecimal character represents 4 bits. 2 hexadecimal characters represent 1 byte. The hexadecimal `0x14` equally translates to decimal number `20`. 0x14 (hex) = 10100 (binary) = 20 (decimal). [Binary-to-hex converter.](https://www.rapidtables.com/convert/number/binary-to-hex.html)

We'll demonstrate how to output the value of a storage slot in hexadecimal format or in bytes32 type using assembly in an upcoming section.

### Primitive and Complex datatype

Throughout this article, our examples will only revolve around primitive datatypes such as unsigned integers (`uint`), integers (`int`), addresses (`address`), and booleans (`bool`).

```solidity
contract PrimitiveTypes {
    uint256 a;
    int256 b;
    address owner;
    bool isTrue;
}
```

These variables occupy at most one storage slot.

Complex datatypes such as structs (`struct{}`), arrays (`array[]`), mappings (`mapping(address => uint256)`), strings (`string`), and bytes (`bytes32`) have a more complicated storage slot allocation. They require a separate article to discuss thoroughly.

## Storage Packing

So far, we've conveniently dealt with `uint256` variables, which span the entire 32 bytes of a storage slot. Other primitive data types, such as `uint8`, `uint32`, `uint128`, `address`, and `bool`, are smaller in size and use less storage space. They can be packed together within the same storage slot.

On a side note, any multiple of 8 up to 256 is a valid `uint`, and `bytes1`, `bytes2`, all the fixed byte sizes `bytes1`, `bytes2`, ... up to `bytes32` are all valid datatypes.

The table below illustrates the storage size of some primitive data types.

|  Type |  Size  |
|---    |     ---|
|`bool`   |1 byte  |
|`uint8`  |1 byte  |
|`uint32` |4 bytes |
|`uint128`|16 bytes|
|`address`|20 bytes|
|`uint256`|32 bytes|

For example, a storage variable of type `address` will require 20 bytes of storage space to store its value, as illustrated in the table above.

```solidity
contract AddressVariable{
    address owner = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;
}
```

In the contract above, `owner` will use up 20 bytes of the 32 bytes available in slot 0 to store its value.

![Storage slot allocation for a single address variable](https://static.wixstatic.com/media/706568_c59675b6711d48d0a1be99a536d42c2c~mv2.png/v1/fill/w_740,h_190,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_c59675b6711d48d0a1be99a536d42c2c~mv2.png)

**Solidity packs variables in storage slots starting from the least significant byte (rightmost byte) and progresses to the left.**

We can verify this by reading the bytes32 representation of the slot:

![Mapping of address owner to its byte sequence](https://static.wixstatic.com/media/706568_cd9feb6ee44c4cc58c9aac57940b47cb~mv2.png/v1/fill/w_740,h_198,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_cd9feb6ee44c4cc58c9aac57940b47cb~mv2.png)

As shown in the diagram above, the value of <span style="color:Orange;">owner</span>, <span style="color:Orange;">0x5B38Da6a701c568545dCfcB03FcB875f56beddC4</span>, is stored starting from the rightmost byte or the least significant byte. The remaining 12 bytes in slot 0 will be unused storage space that another variable can occupy.

**When declared in sequence, smaller sized variables live in the same storage slot if their total size is less than 256 bits or 32 bytes.**

Say we declared a second and a third storage variable of type `bool` (1 byte) and `uint32` (4 bytes) , their values will be stored within the same storage slot as `owner`, slot 0, at the unused storage space.

```solidity
contract AddressVariable {
    address owner = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;

    // new
    bool Boolean = true;
    uint32 thirdvar = 5_000_000;
}
```

<span style="color:#8d28a4;">Boolean</span>, the second declared storage variable, will store its value at the first byte to the left of <span style="color:Orange;">owner</span>'s byte sequence, or, at the least significant byte of the unused storage space. Remember, Solidity packs variables from the right to left.

![Diagram showing the byte sequence that belong to address owner and bool boolean variable](https://static.wixstatic.com/media/706568_1ad9cf209ce34fd7b874f6932ef2e756~mv2.png/v1/fill/w_740,h_201,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_1ad9cf209ce34fd7b874f6932ef2e756~mv2.png)

<span style="color: Green;">uint32 thirdVar</span>, the third storage variable, will store its value to the left of <span style="color: #8d28a4;">Boolean</span>'s byte sequence.

![Mapping of three variables(thirdvar, Boolean, owner) to their respective byte sequence.](https://static.wixstatic.com/media/706568_2b0d75049ae1488c95ea19c73c2aaa52~mv2.png/v1/fill/w_740,h_201,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_2b0d75049ae1488c95ea19c73c2aaa52~mv2.png)

If we were to introduce a fourth storage variable, `address admin`, its value will be stored in the next storage slot, slot 1.

```solidity
contract AddressVariable {
    address owner = 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4;
    bool Boolean = true;
    uint32 thirdVar = 5_000_000;

    // new
    address admin = 0xAb8483F64d9C6d1EcF9b849Ae677dD3315835cb2;
}
```

![Storage slot allocation diagram of 4 state variables.](https://static.wixstatic.com/media/706568_09add3feb5c64e1db65cf69e40b4e55b~mv2.png/v1/fill/w_740,h_132,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_09add3feb5c64e1db65cf69e40b4e55b~mv2.png)

This is because <span style="color:#008aff;">admin</span>'s value **in its entirety** cannot fit into slot 0's unused storage space. There are 7 bytes of storage space left but 20 bytes of consecutive storage space is needed. Therefore, instead of splitting <span style="color:#008aff;">admin</span>'s data between slot 0 and slot 1 (7 bytes in slot 0 and 13 bytes in slot 1), <span style="color:#008aff;">admin</span>'s value will be stored in a new storage slot, **slot 1.**

If a variable's value cannot fit entirely into the remaining space of the current storage slot, it will be stored in the next available slot.

### Declare smaller variables together

```solidity
uint16 public a;
uint256 public x; // uint256 in the middle
uint32 public b;
```

In this arrangement, `uint16 a` and `uint32 b` **will not** be packed together.

Instead, `a` will be stored at slot 0, `x` at slot 1, and `b` at slot 2, using up three storage slots. The storage slot allocation would look like the diagram below:

![inefficient storage slot allocation for a uint256 variable declared in between two smaller storage variables](https://static.wixstatic.com/media/706568_2796450a514943efa19e2eea4b986099~mv2.png/v1/fill/w_740,h_343,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_2796450a514943efa19e2eea4b986099~mv2.png)

A better practice is to reorder the declarations to allow the smaller datatypes to be packed together.

```solidity
uint256 public x;
// packed together
uint16 public a;
uint32 public b;
```

This configuration allows <span style="color:Red">a</span> and <span style="color:Green">b</span> to share a storage slot, thereby optimizing storage space.

![Efficient storage slot allocation for three storage variables](https://static.wixstatic.com/media/706568_3fc7736a7d974e738cd5b74810cba347~mv2.png/v1/fill/w_740,h_260,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_3fc7736a7d974e738cd5b74810cba347~mv2.png)

Now that we've understood the theory behind how primitive variables are kept in storage, we are finally ready to learn how to manipulate them in assembly, using YUL.

## Storage Slot Manipulation in Assembly (YUL)

Low level assembly (Yul) gives a higher degree of freedom in performing storage related operations. It allows us to directly read and write from individual storage slots and access a storage variable's properties.

There are two opcodes related to storage in Yul: `sload()` & `sstore()`.

- `sload()` reads the value stored by a specific storage slot.
- `sstore()` updates the value of a specific storage slot with a new value.

Two other important Yul keywords are `.slot` and `.offset`.

- `.slot` returns the location within the storage slots.
- `.offset` returns the byte offset of the variable. (This will be discussed in Part 2)

### The `.slot` keyword

The contract below contains three uint256 storage variables.

```solidity
contract StorageManipulation {
    uint256 x;
    uint256 y;
    uint256 z;
}
```

You should be able to deduce that `x`, `y` and `z` store their values in slot 0, slot 1 and slot 2, respectively. We can prove this by accessing the storage variable's property using the `.slot` keyword.

**`.slot`** **tells us at which storage slot a variable keeps its value.**

For example, to query `x`'s storage slot, append `.slot` to the variable name: `x.slot` in assembly.

```solidity
function getSlotX() external pure returns (uint256 slot) {
    assembly {// yul
        slot := x.slot // returns slot location of x
    }
}
```

`x.slot` returns a value of **0** , which corresponds to the storage slot where `x` stores its state—**slot 0**.

![x.slot returns the slot number of the storage variable x](https://static.wixstatic.com/media/706568_6b7b58f6c2c74e52a9f744aa22412242~mv2.png/v1/fill/w_740,h_265,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_6b7b58f6c2c74e52a9f744aa22412242~mv2.png)

`y.slot` will return **1** , which corresponds to `y`'s storage slot— **slot 1**.

![y.slot returns the slot number of the storage variable y](https://static.wixstatic.com/media/706568_2e593f6b58b64f8e8b3e2f37424d5f33~mv2.png/v1/fill/w_740,h_341,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_2e593f6b58b64f8e8b3e2f37424d5f33~mv2.png)

`z.slot` will return **2** , which corresponds to `z`'s storage slot— **slot 1**.

![z.slot returns the slot number of the storage variable z](https://static.wixstatic.com/media/706568_cb087c6f34b94e4cb8f5de0e8f2162cf~mv2.png/v1/fill/w_740,h_421,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_cb087c6f34b94e4cb8f5de0e8f2162cf~mv2.png)

### Reading the value of variables directly from their storage slot: `sload()`

Yul allows us to read the value stored by individual storage slots. The `sload(slot)` opcode is used for this purpose. It requires one input, `slot`, the storage slot identifier and returns the entire 256 bit of data stored at the specified slot location.

The slot identifier can be either the `.slot` keyword (`sload(x.slot)`), a local variable (`sload(localvar)`) or a hardcoded number (`sload(1)`).

Here are a few examples on how to use the `sload()` opcode:

```solidity
contract ReadStorage {

    uint256 public x = 11;
    uint256 public y = 22;
    uint256 public z = 33;

    function readSlotX() external view returns (uint256 value) {
        assembly {
            value := sload(x.slot)
        }
    }

    function sloadOpcode(uint256 slotNumber)
        external
        view
        returns (uint256 value)
    {
        assembly {
            value := sload(slotNumber)
        }
    }
}
```

The function `readSlotX()` retrieves the 256 bit data stored in `x.slot` (slot 0) and returns it in `uint256` format, which equals 11.

```solidity
function readSlotX() external view returns (uint256 value) {
    assembly {
        value := sload(x.slot)
    }
}
```

- `sload(0)` reads from slot 0, which stores the value of 11.
- `sload(1)` reads from slot 1, which stores the value of 22.
- `sload(2)` reads from slot 2, which stores the value of 33.
- `sload(3)` reads from slot 3, which stores nothing, it is still in its default state.

The animation below visualizes how the `sload` opcode functions.

<video src="https://video.wixstatic.com/video/706568_665a627d4bce4eb1bfb7a340f059937a/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

The function `sloadOpcode(slotNumber)` allows us to read the value of any arbitrary storage slot. It then returns the value in uint256 format.

```solidity
function sloadOpcode(uint256 slotNumber)
    external
    view
    returns (uint256 value)
{
    assembly {
        value := sload(slotNumber)
    }
}
```

Notably, **`sload()`** **does not perform a type check.**

In Solidity, we cannot return a uint256 variable in bool format as it will incur a type error.

```solidity
function returnX() public view returns (bool ret) {
    // type error
    ret = x;
}
```

But if the same set of operation is performed in Yul, the code will still compile.

```solidity
function readSlotX_bool() external view returns(bool value) {
    // return in bool
    assembly{
        value:= sload(x.slot) // will compile
    }
}
```

We'll discuss why this is possible in detail in Part 2. To give you a rough idea, in assembly, every variable is essentially treated as a `bytes32` type. Outside of the assembly scope, the variable will resume its original type and format the data accordingly.

Consequently, we can use this property to examine the value of a storage slot in bytes32 format.

```solidity
contract ReadSlotsRaw {
    uint256 public x = 20;

    function readSlotX_bool() external view returns (bytes32 value) {
        assembly {
            value := sload(x.slot) // will compile
        }
    }
}
```

![Visual explanation of returning the value of a storage slot in bytes32](https://static.wixstatic.com/media/706568_e6ae45a9d9924157858372d1ce3f994f~mv2.png/v1/fill/w_740,h_738,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_e6ae45a9d9924157858372d1ce3f994f~mv2.png)

### Writing to a storage slot using the `sstore()` opcode

Yul gives us direct access to modify the value of a storage slot using the `sstore()` opcode.

`sstore(slot, value)` stores a 32-byte long value directly to a storage slot . The opcode takes two parameters, **slot** and **value:**

- `slot`: This is the targeted storage slot which we are writing to.
- `value`: The 32-byte value to be stored at the specified storage slot. If the value is less than 32 bytes, it will be left padded with zeroes

`sstore(slot, value)` overwrites the entire storage slot with a new value.

The contract below demonstrates how to use `sstore()`; we use it to change the values of `x` and `y`:

```solidity
contract WriteStorage {
    uint256 public x = 11;
    uint256 public y = 22;
    address public owner;

    constructor(address _owner) {
        owner = _owner;
    }

    // sstore() function

    function sstore_x(uint256 newval) public {
        assembly {
            sstore(x.slot, newval)
        }
    }

    // normal function
    function set_x(uint256 newval) public {
        x = newval;
    }
}
```

`sstore_x(newVal)` directly updates the value stored in the storage slot that `x` references, effectively changing the value of `x`. The animation below visualizes what happens when we call the opcode `sstore_x(88)`.

<video src="https://video.wixstatic.com/video/706568_c3a727877521411a97f6233fdb0a531e/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

Both `sstore_x(newVal)` and `set_x()` perform the same function: They update the value of `x` with a new value.

The function below, `sstoreArbitrarySlot(slot, newVal)`, is capable of changing the value of any storage slot, therefore, it is advised to never put this in production.

```solidity
function sstoreArbitrarySlot(uint256 slot, uint256 newVal) public {
    assembly {
        sstore(slot, newVal)
    }
}
```

Calling `sstoreArbitratySlot(`**`1 , 48`**`)`, will change the value of `y` from `22` to `48`. Since `y` keeps its value at storage slot 1, it overrides the value of 22 in slot 1 and changes it to 48.

**`sstore()` also does not type check.**

Normally, when we try to assign an `address` type to a `uint256` type, it would return a type error and the contract would not compile:

```solidity
address public owner;

function TypeError(uint256 value) external {
    owner = value; // ERROR: Type uint256 is not implicitly convertible to expected type address.
}
```

```
ERROR: Type uint256 is not implicitly convertible to expected type address.
```

This error will not trigger with `sstore()` as it does not perform a type check.

```solidity
contract WriteStorage {
    address public owner;

    function sstoreOpcode(uint256 value) public {
        assembly {
            sstore(owner.slot, value)
        }
    }
}
```

## Manipulating storage packed variables in Yul Part 2

`sstore` and `sload` operate on lengths of 32 bytes. This is convenient when dealing with `uint256` type as the entire 32 bytes read or written correspond directly to the `uint256` variable. However, the situation becomes more complex when dealing with variables that are packed within the same storage slot. Their byte sequence occupies only a portion of the 32 bytes and in assembly, we do not have an opcode to directly modify or read from their byte sequence in storage.

In Part 2, we will cover manipulating storage-packed variables in Yul using bit-manipulation and bit-masking techniques.

*Originally Published July 15, 2024*
