# Understanding ABI encoding for function calls

ABI encoding is the data format used for making function calls to smart contracts. It is also how smart contracts encode data when making calls to other smart contracts.

This guide will show how to interpret ABI encoded data, how to compute the ABI encoding, and teach the relationship between the function signature and the ABI encoding.

Let's get into it…

## Solidity abi.encodeWithSignature and low level calls

If we were to make a [low level call](https://www.rareskills.io/post/low-level-call-solidity) to another smart contract with a public function `foo(uint256 x)` (passing `x = 5` as the argument), we would do the following:

```solidity
otherContractAddr.call(abi.encodeWithSignature("foo(uint256)", (5));
```

We can look at the actual data returned by `abi.encodeWithSignature("foo(uint256)", (5))` with the following code:

```solidity
function seeEncoding() external pure returns (bytes memory) {
    return abi.encodeWithSignature("foo(uint256)", (5)); 
}
```

and we would get the following result (which is ABI encoded):

```solidity
0x2fbebd380000000000000000000000000000000000000000000000000000000000000005
```

Interpreting and understanding this data like this is the goal of this article.

## The key components of an ABI encoded function call

An encoded ABI function call is the concatenation of the function selector and the encoded arguments to the function (if the function takes arguments).

### The function signature

The function signature is the combination of the function name and its argument types without spaces.   

For example, the function signature for the function below:

```solidity
function transfer(address _to, uint256 amount) public {

//

}
```

is `transfer(address,uint256)`. Note that you must use the full argument data types, such as `uint256` instead of `uint`. Also, the variable names like the `_to` and `amount` are not part of the function signature. It is also important that there are no spaces in the string such as `transfer(addres, uint256)`.

Per the [Solidity documentation](https://docs.soliditylang.org/en/v0.8.26/abi-spec.html#mapping-solidity-to-abi-types), are some "corner cases" to be aware of when computing the function signature:

- Structs are treated like tuples
- Payable addresses, interfaces, and contract types are treated as addresses
- The "memory" and "calldata" modifiers are ignored
- An enum is uint8
- A user defined type is treated as its underlying type
    
### The function selector

The [function selector](https://www.rareskills.io/post/function-selector) is simply the first 4 bytes of the [Keccak-256](https://eth-hash.readthedocs.io/en/stable/) hash of a function signature that Solidity uses to identify a function. For example, the Keccak-256 hash of our previously mentioned function signature `transfer(address,uint256)` is this hexadecimal value:

```solidity
0xa9059cbb2ab09eb219583f4a59a5d0623ade346d962bcd4e46b11da047c9049b
```

However, only the first 4 bytes of the hash result `0xa9059cbb` is used to identify the function; those four bytes are the function selector.

You can use the ethers JavaScript library to convert the `transfer()` function signature to its selector as shown below:

```javascript
const ethers = require('ethers'); // Ethers v6
const functionSignature = 'transfer(address,uint256)';
const functionSelector = ethers.id(functionSignature).substring(0, 10)
console.log(functionSelector);
```

The result would be like this:

![Javascript Function to convert a function signature to a function selector](https://static.wixstatic.com/media/706568_7b5c62730e5e49fbb8572929d6d3f112~mv2.png/v1/fill/w_740,h_280,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_7b5c62730e5e49fbb8572929d6d3f112~mv2.png)

In Solidity, this function computes the function selector:

```solidity
function getSelector() public pure returns (bytes4 ret) {
    return bytes4(keccak256("transfer(address,uint256)")); // 0xa9059cbb
}
```

You can also use this [keccak256 conversion website](https://emn178.github.io/online-tools/keccak_256.html?input_type=utf-8&input=transfer(address%2Cuint256)) to see the conversion without writing any code:

![Keccak-256 converter website snippet](https://static.wixstatic.com/media/706568_ad74435a97454610854f6dddb76799a3~mv2.png/v1/fill/w_740,h_564,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/706568_ad74435a97454610854f6dddb76799a3~mv2.png)

Now that we have a clear understanding of what the function selector is, let's consider the next component of an ABI encoding for function calls — function inputs or arguments.

### Function inputs or arguments

When calling a function that takes no arguments, the function selector alone will be all the encoding needed to call the function. For example, the function `play()` will be identified by its function selector `0x93e84cd9` and that will be the entire data needed.

However, it gets complex if the function takes arguments, such as `transfer(address to, uint256 amount)`, then the function arguments must be ABI encoded and concatenated to the function selector.

Let's use `transfer(address to, uint256 amount)` as a running example to help us understand how the argument encoding is done:

```solidity
function transfer(address to, uint256 amount) public {
	//
}
```

This data for the function call isn't stored permanently within the function or the contract itself. Instead, it lives in a space called "calldata." You cannot modify the data in calldata, as it's created by the transaction sender and then becomes read-only.

You can see the calldata of a transaction in Etherscan. Below is a screenshot showing an example of [transfer transaction calldata](https://etherscan.io/tx/0xb88e8b39284ef7a66daaee9443792fcef6f9b9ba3e2ca14e37355cb8a8069d0e) for an example [ERC-20 token](https://etherscan.io/token/0x8DE5B80a0C1B02Fe4976851D030B36122dbb8624):


![Transaction calldata on Etherscan](https://static.wixstatic.com/media/706568_a0b16ef1ab084dd599c6efa39d4d957a~mv2.png/v1/fill/w_740,h_373,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_a0b16ef1ab084dd599c6efa39d4d957a~mv2.png)

Etherscan refers to the function selector as `MethodID` below the function description (see the <span style="color: Red;">red box</span> in the screenshot below). So the `methodID` for `transfer()` is `0xa9059cbb`.

![Function selector and signature shown in etherscan](https://static.wixstatic.com/media/706568_1d3bd902a6da4f8ea23a99a93af431a5~mv2.png/v1/fill/w_740,h_314,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_1d3bd902a6da4f8ea23a99a93af431a5~mv2.png)

Then what follows are two long hexadecimal values tagged as `[0]` and `[1]`. Those hexadecimal values represent the two input data arguments: the `_to`, `address` and the `_value`, `uint256`.

Etherscan helps us separate and interpret the calldata information into 32-bytes (64 characters) per line. However, the actual calldata will be bundled together and sent as one long string and it should look like the one below:

```solidity
0xa9059cbb000000000000000000000000f89d7b9c864f589bbf53a82105107622b35eaa4000000000000000000000000000000000000000000000028a857425466f800000
```

You can also see the actual complete calldata in its original format on Etherscan by clicking on the "View input as" and selecting the "Original" option as shown below:

![Original option for reading calldata on Etherscan](https://static.wixstatic.com/media/706568_6ead892408214befaf9106defda25428~mv2.png/v1/fill/w_740,h_185,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_6ead892408214befaf9106defda25428~mv2.png)

To better understand what's happening under the hood, let's break down the calldata and extract the relevant information of the transaction.

## Dividing the calldata

Let's consider the calldata below and identify its components:

```
0xa9059cbb0000000000000000000000003f5047bdb647dc39c88625e17bdbffee905a9f4400000000000000000000000000000000000000000000011c9a62d04ed0c80000
```

First, we need to know the function signature — without that we cannot decode the data. So, here is the function signature for the above calldata:

```solidity
transfer(address,uint256)
```

We then put the hexadecimal notation (0x) and the function selector on their own line. The function selector is always 4 bytes (8 hex characters). Finally, we divide each of the following 32 bytes into their own line. As we will see later, Solidity encodes the data in 32-byte increments.

```solidity
0x <---------- Hexadecimal notation
a9059cbb <---- Function selector
0000000000000000000000003f5047bdb647dc39c88625e17bdbffee905a9f44
00000000000000000000000000000000000000000000011c9a62d04ed0c80000
```

### Function selector

The function selector is the first 4 bytes of the calldata `0xa9059cbb`.

![First 4 bytes of a calldata represents the function selector](https://static.wixstatic.com/media/706568_991bd55d1b844a0cb032c25634d7dc4b~mv2.png/v1/fill/w_740,h_158,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_991bd55d1b844a0cb032c25634d7dc4b~mv2.png)

### Address

The address in `transfer(address,uint256)` is the next 32 bytes value. The actual address is 20 bytes but it is left padded with leading zeros to make it 32 bytes.

Address:   
`000000000000000000003f5047bdb647dc39c88625e17bdbffee905a9f440`

![The 32 bytes of calldata that represents the address parameter](https://static.wixstatic.com/media/706568_5a2893dfaf62480f92150977729b117d~mv2.png/v1/fill/w_740,h_153,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_5a2893dfaf62480f92150977729b117d~mv2.png)

Essentially, the receiver address will be the value above but without the extra zero paddings: i.e. `0x3F5047BDb647Dc39C88625E17BDBffee905A9F44`.

### Amount

And finally, the last item in `transfer(address,uint256)` is the amount. The amount (`0000000000000000000011c9a62d04ed0c80000`) is left-padded with leading zeros to become 32 bytes as shown below:

![32 bytes portion of the calldata that represents the amount parameter](https://static.wixstatic.com/media/706568_d06365d7c9634d9e8398ec014c4d8b1f~mv2.png/v1/fill/w_740,h_144,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_d06365d7c9634d9e8398ec014c4d8b1f~mv2.png)

Here is a Python snippet to help you quickly convert hexadecimal to decimal:

```python
>>> int("0x11c9a62d04ed0c80000", 16)
5250000000000000000000
```

And we can also convert the decimal back to hexadecimal as shown below:

```python
>>> hex(5250000000000000000000)
0x11c9a62d04ed0c80000
```

### Data types and padding

We've established that each item in calldata is encoded as a 32-byte word and padded with zeros if the item doesn't take up the entire 32-byte word.   

As a [rule](https://docs.soliditylang.org/en/v0.8.25/internals/layout_in_calldata.html), every fixed-size data type such as `int`, `bool`, and `uint` of all sizes (unit8-uint256) will be encoded as a 32 bytes word padded to the left with zeos if needed.

For example, if you have a `uint8` with a value of `5`, it will be encoded as   
`0x0000000000000000000000000000000000000000000000000000000000000005`.

Similarly, a `bool` with a `true` value would also be left padded and encoded as `0x0000000000000000000000000000000000000000000000000000000000000001`.

However, the dynamic sized data types `bytes` and `string` are right padded. For example, the bytes `0x68656c6c6f` which represents `hello` will be encoded as a 32-bytes word padded with zeros to the right `0x68656c6c6f000000000000000000000000000000000000000000000000000000`.

Fixed-sized data types in Solidity include:    
- bool
- uints
- bytes of fixed size (byteN)
- address
- tuple, struct with fixed data
- fixed-size array
    
Below are the dynamic data types in Solidity:   
- bytes
- string
- dynamic array
- a fixed-size array that contains dynamic types
- a struct that contains any of the above dynamic types

## Working with dynamic calldata

So far, our focus has been on static calldata argument types like `address` and `uint256`. While static types are fairly straightforward to encode, encoding arrays and strings can be a bit complicated due to the varying size of data they hold.

Let's consider a function that takes an array of uints and a single address. While the implementation details for our fuction are not relevant here, the function signature should look like this:

```solidity
transfer(uint256[],address)
```

Now, let's move our focus to encoding the array. Let's assume we are passing the following data to the transfer function:

```solidity
transfer([5769, 14894, 7854], 0x1b7e1b7ea98232c77f9efc75c4a7c7ea2c4d79f1)
```

This is the calldata for our example function above with the signature `transfer(uint256[],address)`. Let's examine it and see how every part is encoded following the pattern we described above.

First, we'll start with encoding the "offset" of the array `uint256[]` but what is an offset?

![Highlight of the array argument of an ABI encoded data](https://static.wixstatic.com/media/706568_3d476cde32064315887b319194459678~mv2.png/v1/fill/w_740,h_246,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_3d476cde32064315887b319194459678~mv2.png)

### Offset

The offset is used to locate within the calldata where specific dynamic data starts or can be found.

Following our example, we have one dynamic datatype `uint256[]` and a static type `address`. The offset of `uint256[]` in the above calldata is 40 in hexadecimal (64 in decimal) and its encoding takes up 32 byte words.

Since the dynamic array is the first argument to the function, the offset is the first 32-byte word in the calldata:

![highlight of the offset of an abi encoded data](https://static.wixstatic.com/media/706568_71ccf02e51cb43edb5e94b9b268bdd15~mv2.png/v1/fill/w_740,h_246,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_71ccf02e51cb43edb5e94b9b268bdd15~mv2.png)

To further explain how the offset works, the image above highlights where the offset of the array is located in the calldata. Each byte word is numbered from

- 0-31 (the first row of 32 bytes)
- 32-63 (the second row of 32 bytes)
- 64-95 (the third row of 32 bytes)
- etc

So 64 (40 hex) is the leftmost byte (pair of hex characters) on the third row where the <span style="color: Green;">green</span> highlight ends. This is where the offset is pointing.

In this example, the offset is the distance from the start of the first byte after the function selector to where the dynamic data (array) starts. **We will see later however, that the offset does not always mean "offset from the first byte after the function selector."**

### Encoding the static data — the address

The next line is the address, which is a static 32-byte word padded with leading zeros. It's the same address we already passed in, since the address is already in hexadecimal format.

![Encoding the address in abi format](https://static.wixstatic.com/media/706568_e8adc195158744c18849462133adc8ba~mv2.png/v1/fill/w_740,h_280,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_e8adc195158744c18849462133adc8ba~mv2.png)

### Encoding the length of a dynamic data — the array

The next line is the length of the array, which is the number of items in the array. As you can see, we have 3 items in the array: `[5769, 14894, 7854]`. The length of the array is 3 as represented in the image below:

![Highlighted length of array within the ABI encoded data](https://static.wixstatic.com/media/706568_a9b7651fbd604ba9864a225eed230160~mv2.png/v1/fill/w_740,h_284,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_a9b7651fbd604ba9864a225eed230160~mv2.png)

### Hex encoding array elements

So far, we've encoded the static type, the offset, and the length of the array. Next, let's encode the actual array elements. Each element of the array will be represented as a hexadecimal number like the image below:

![Converting Decimal to Hex abi encoding](https://static.wixstatic.com/media/706568_8ce88ad341134aec8db732568a1c65b0~mv2.png/v1/fill/w_740,h_410,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_8ce88ad341134aec8db732568a1c65b0~mv2.png)

We convert each of the integers to their hexadecimal representation and add leading zeros to them. So, the array items will be the following:

![Comparing ABI encoded data to the invoked function parameters](https://static.wixstatic.com/media/706568_df36b9fd302c4789879e879a5ebb2ad8~mv2.png/v1/fill/w_740,h_311,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_df36b9fd302c4789879e879a5ebb2ad8~mv2.png)

This completes our discussion of the calldata of this ABI encoding.

The following video summarizes everything we learned about how to encode calldata for `transfer(uint[], address)`:

<video src="https://video.wixstatic.com/video/706568_29bfe4ea94b94cf49a0bd42a0e4d9f44/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

### ABI encoding a string argument

Encoding a string is straightforward, you only need to encode the following:

- the offset
- the length of the string
- the content of the string (UTF-8 encoded)
    
Here is an example using a function that contains a string argument:

```solidity
play(string)
```

When we pass in a value:

```solidity
play("Eze")
```

the calldata will be the text below:

```solidity
0x
718e6302
0000000000000000000000000000000000000000000000000000000000000020
0000000000000000000000000000000000000000000000000000000000000003
457a650000000000000000000000000000000000000000000000000000000000
```

The offset is represented as 20 in hexadecimal, as the position of the string encoding is just 32 bytes (32 in decimal is 20 in hexadecimal) away from the beginning of the calldata after the function selector. We can also see that the length of the string (<span style="color: Red;">Eze</span>) is 3 as there are just 3 characters in the string, one byte each (one byte is two hex chars).

![Abi encoded data of a string arguement](https://static.wixstatic.com/media/706568_ed764eb8831a436ba5c8675f678c5766~mv2.png/v1/fill/w_740,h_148,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_ed764eb8831a436ba5c8675f678c5766~mv2.png)

The string "Eze" is only using ASCII characters which take one byte each (hence a length of 3). However, unicode characters like "好" take up 3 bytes. The [maximum size a utf-8 character can be is 4 bytes](https://stackoverflow.com/questions/9533258/what-is-the-maximum-number-of-bytes-for-a-utf-8-encoded-character). The string "你好" has a length of 6 bytes.

## Encoding structs/tuples in a calldata

Tuples and structs are encoded identically because [structs are mapped](https://docs.soliditylang.org/en/latest/abi-spec.html#mapping-solidity-to-abi-types) to the ABI type `tuple`.

According to the Solidity ABI encoding specification, the encoding of a struct is the concatenation of the encoding of its members, with the static types padded to 32 bytes.

Suppose we have the following contract:

```solidity
contract C {

    struct Point {
        uint256 x;
        uint256 y;
    }

    function foo(Point memory point) external pure {
        //...   
    }
}
```

The function signature of `foo` would be `foo((uint256, uint256))`. This is no different than if it took a tuple as an input. If it took a dynamic array of points (structs), the function signature would be `foo((uint256, uint256)[])`.

If a struct's element are all fixed-size data, we'll encode the entire struct as static type and there will be no need for an offset. However, the encoding of the struct will change if it has a dynamic-sized data type as at least one of its field.

For example, a struct like this one below:

```solidity
RareToken {
    uint256 n;
}

send(RareToken,address)
```

will be encoded as a static type if we pass the following arguments to it `send( RareToken(1), 0x1b7e1b7ea98232c77f9efc75c4a7c7ea2c4d79f1))` as demonstrated in the next image:

![ABI encoding of tuples](https://static.wixstatic.com/media/706568_5e14354813454064b3cdfeb3bee98b0e~mv2.png/v1/fill/w_740,h_280,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_5e14354813454064b3cdfeb3bee98b0e~mv2.png)

However, if there is a dynamic type in the struct, we'll have to encode the struct as a dynamic type. Let's take this one as an example:

```solidity
RareToken {
    uint256;
    string;
}

send(RareToken)
```

The encoding of the function above would be the code in the image below if we pass the following data to it `send(RareToken(50,"Eze"))` . The struct contains one dynamic type and a static type as shown in this diagram:

![Offset of string in struct](https://static.wixstatic.com/media/935a00_68b800fc0d404a278a1536199dd03a3f~mv2.png/v1/fill/w_740,h_188,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_68b800fc0d404a278a1536199dd03a3f~mv2.png)

### ABI encoding multiple struct arguments

Now, suppose our function `send()` took 3 structs as arguments instead of one. The calldata would be as follows:

![abi encoding multiple struct arguements](https://static.wixstatic.com/media/706568_5a8bdcc8d3d64575953314e37e5f0ddc~mv2.png/v1/fill/w_740,h_344,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_5a8bdcc8d3d64575953314e37e5f0ddc~mv2.png)

The first three 32-byte words are offsets because the function takes 3 arguments as input, and each of those arguments are dynamic data types (structs with a dynamic type field).

The column of 0x00, 0x20, …, 0x1c0 on the left shows how the offset points to the location in calldata. Note that the offset starts from the first offset, not from where the offset is located. We'll explore offsets more when we look at nested dynamic data.

## Encoding a fixed sized array with static type

Encoding fixed size arrays depends on the content in the array, if the fixed size array contains dynamic types then the fixed size array will be encoded as a dynamic type. If it contains only static types, it will be treated as a static type an encoded as such. This follows the same logic as a struct that contained dynamic data as shown in the above section.

Let's start with a fixed size array of only static types with length of 3:

```solidity
play(uint256[3])
```

and pass this data to it:

```solidity
play([1,2,3])
```

And here is the encoding of the array:

![ABI encoding of a fixed sized array with static type](https://static.wixstatic.com/media/706568_98a79a4346214d0bb1c7e3a324cf62b2~mv2.png/v1/fill/w_740,h_149,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_98a79a4346214d0bb1c7e3a324cf62b2~mv2.png)

As you can see there are no offsets in the calldata. It's just the encoding of the elements of the array.

## Encoding a fixed sized array holding a dynamic type

Let's consider a scenario where the fixed size array contains dynamic data. The function below will contain an array of two strings.

```solidity
plays(string[2])
```

If we pass the following strings to it:

```solidity
play(["Eze","Sunday"])
```

We'll get this calldata when encoded:

![ABI encoding a fixed sized array of dynamic type members](https://static.wixstatic.com/media/706568_a0ccde8f605a4607b51cdd07328464f5~mv2.png/v1/fill/w_740,h_222,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_a0ccde8f605a4607b51cdd07328464f5~mv2.png)

Because the fixed size array was an array of dynamic data types, its entirety was encoded as a dynamic array, the only difference is that the length of the array was not encoded because the function signature defined it as a fixed length array of 2. If we consider the same function but with dynamic length:

```solidity
plays(string[])
```

we'll notice that the length of the array will be encoded also:

![Length of array highlighted within the abi encoded data](https://static.wixstatic.com/media/706568_a3960bbe6a764738a50e4bc93ac32871~mv2.png/v1/fill/w_740,h_220,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_a3960bbe6a764738a50e4bc93ac32871~mv2.png)

## Multiple array arguments and nested arrays in calldata

Working with multiple and nested arrays in calldata can be a little complex and tricky. However, the general pattern remains similar. In this section, we'll learn how to encode and decode nested arrays and get a better intuition for how the offset works.

We'll use the following function signature as an example:

```solidity
transfer(uint256[][],address[])
```

Let's also pass the following data as arguments to the function:

```solidity
transfer([[123, 456], [789]], [0x5B38Da6a701c568545dCfcB03FcB875f56beddC4, 0x7b38da6a701c568545dcfcb03fcb875f56bedfb3])
```

So, the calldata for this function and the argument will be the hexadecimal below:

```solidity
0x7a63729a
0000000000000000000000000000000000000000000000000000000000000040
0000000000000000000000000000000000000000000000000000000000000140
0000000000000000000000000000000000000000000000000000000000000002
0000000000000000000000000000000000000000000000000000000000000040
00000000000000000000000000000000000000000000000000000000000000a0
0000000000000000000000000000000000000000000000000000000000000002
000000000000000000000000000000000000000000000000000000000000007b
00000000000000000000000000000000000000000000000000000000000001c8
0000000000000000000000000000000000000000000000000000000000000001
0000000000000000000000000000000000000000000000000000000000000315
0000000000000000000000000000000000000000000000000000000000000002
0000000000000000000000005b38da6a701c568545dcfcb03fcb875f56beddc4
0000000000000000000000007b38da6a701c568545dcfcb03fcb875f56bedfb3
```

The new feature of our `transfer()` function is that it has two arrays, and one of those arrays has two sub-arrays.

Here's how a calldata structure for a nested and multiple array is constructed on a high level:

- Offsets. It starts with defining the offsets for the different locations of the arrays. Assuming there are multiple array arguments, the encoding of the offsets of those arrays will be defined first. For our example, there will be two offsets.
    - Offset to the first dynamic type
    - Offset to the second dynamic type
    - Offset to the `nth` dynamic type (where applicable)
- The length of the first array parameter (2 in our example: `[[123, 456], [789]]`) comes next and lives where the first offset points. This is the length of the *entire* array. Before you start working on each array, you begin with defining its length. So for every sub-array, you'll define their length (later in the ABI encoding).
- Next, encode the sub-arrays of the first array parameter
    - Offset to the first sub-array (`[123, 456]`)
    - Offset to the second sub-array (`[789]`)
        - Length of the first sub-array (2 in our example)
            - First item of the first sub-array(`123`)
            - Second item of the first sub-array(`456`)
        - Length of the second sub-array (1 in our example)
            - First item of the second sub-array(`789`)        
- Once you are done with all the items of the first array parameter, start encoding the next array parameter
    - Length of the second parameter array (2 addresses in our example)
    - and elements of the second parameter (the addresses in our example), if the second array parameter has sub-arrays, you follow the same pattern as described above.

Now, let's visualize the calldata of our example.

First, arrange it in 32 bytes (64 characters) per line, aside from the `0x` and the function selector, to make it easier to read. The function signature and calldata are at the top of the image:

![Calldata for nested array and dynamic array](https://static.wixstatic.com/media/706568_1004767d56594cc6907fe3391e75062a~mv2.png/v1/fill/w_740,h_427,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_1004767d56594cc6907fe3391e75062a~mv2.png)

### Offset of the first array argument

The first 32-byte word of the calldata string is the offset, indicating where the data for the first array parameter starts. Here is a visual representation:

![Offset of the nested array parameter within the calldata](https://static.wixstatic.com/media/706568_d8ad08079f6d4855ab74d4fffff9c2aa~mv2.png/v1/fill/w_740,h_325,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_d8ad08079f6d4855ab74d4fffff9c2aa~mv2.png)

### Offset of the second array argument

The next step is the encoding of the offset for the second array argument.

As we saw with the example of multiple dynamic structs, this offset does not "start counting" from the location of the offset, but from the first offset.

Offsets do not in general "start counting" from their current location, but from the first offset that described that "level" of the nested data structure. This concept will become more clear as we explore the sub arrays.

The image below highlights how the second offset points to the second argument data (the array of addresses). Note that the offset is pointing to a "2" as the second argument is an array of two addresses.

![Offset of the second array argument within the calldata](https://static.wixstatic.com/media/706568_e9d892ac798847b3803ee82646cfbc14~mv2.png/v1/fill/w_740,h_258,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_e9d892ac798847b3803ee82646cfbc14~mv2.png)

Since each byte word is numbered from:

- 0-31
- 32-63
- 64-95
- etc
    
320 bytes (140 hex) is the leftmost 0 on the row where the highlight ends.

### Length of the first array

Next, we need to encode the length of the first array. The sub-array items include `[123, 456]` and `[789]`, which forms `[[123, 456], [789]]`. Since there are two nested arrays, the length is 2. The length is represented in the calldata as shown below:

![Length of the first array argument within the callda](https://static.wixstatic.com/media/706568_9e2a3fe2dea34d7392853df106d852e5~mv2.png/v1/fill/w_740,h_346,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_9e2a3fe2dea34d7392853df106d852e5~mv2.png)

### Offsets of the sub-arrays in the first array argument

After the length of the array, we have the offsets that shows where the content of those arrays are stored. There are two offsets, since there are two subarrays: `[123, 456]` and `[789]`.

#### Offset of the first sub-array

The offset of the first sub array(`[123,456]`) is the `40` pointed to by the box on the right. Both of them "start counting" from the first word after the length that defines the array. Again, note that it is pointing to a word holding a `2` because `[123,456]` is of length 2.

![Offset of the first sub array within the calldata](https://static.wixstatic.com/media/706568_780a9b42e3e64a01897d2a13e82bfe8f~mv2.png/v1/fill/w_740,h_268,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_780a9b42e3e64a01897d2a13e82bfe8f~mv2.png)

#### Offset of the second sub-array

The offset of the second sub-array (`[789]`) doesn't start from the beginning of the calldata. It's located at `a0` (the last <span style="color: Red;">red</span> highlight in the image below) which is 160 bytes (in decimal) from where the first sub-array offset is declared.

Recall that offsets do not in general "start counting" from their current location, but from the first offset that described that "level" of the nested data structure. We are now one level deep in the nested array, so our first offset is the 40 highlighted in purple below. This offset does not start counting from its own location:

![offset of second sub array within the calldat](https://static.wixstatic.com/media/706568_8ef2dd3ce679449aac2310eff9050b36~mv2.png/v1/fill/w_740,h_256,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_8ef2dd3ce679449aac2310eff9050b36~mv2.png)

### The length of the first sub-array

The length of the first sub-array comes next. The first sub-array contains 2 items and is identified by the yellow highlight below:

![Length of first sub array within the calldata](https://static.wixstatic.com/media/706568_c96e1347efda4d138204e7717d7273b7~mv2.png/v1/fill/w_740,h_338,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_c96e1347efda4d138204e7717d7273b7~mv2.png)

### The items in the first sub-array

The next two words are the hex representations of the two items in the first sub-array as shown in the image below:

![Items of the first sub-array within the calldata](https://static.wixstatic.com/media/706568_619c40b0804a4ab99ec000afedf8c6bd~mv2.png/v1/fill/w_740,h_375,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_619c40b0804a4ab99ec000afedf8c6bd~mv2.png)

### The length of the second sub-array

Then we move to the length of the second sub-array which only holds one item:

![Length of the second sub-array within the calldata](https://static.wixstatic.com/media/706568_95412cfb50044d0cb3dbcf401296565f~mv2.png/v1/fill/w_740,h_359,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_95412cfb50044d0cb3dbcf401296565f~mv2.png)

### The item in the second sub-array

The second array has only one item, as you can see from its length. We also represent the value of the single item in the array (315), which is the hex value of `789`, in the next 32 bytes, as shown below.

![The item of the second sub-array within the calldata](https://static.wixstatic.com/media/706568_4511072399e74bd89a3e9b9a5ff9d3bf~mv2.png/v1/fill/w_740,h_391,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_4511072399e74bd89a3e9b9a5ff9d3bf~mv2.png)

### Length of the second array

Finally, we get to represent the length of the second parameter, the address array. We have 2 addresses, so the length is 2; and the two parameters are the addresses represented in this diagram.

![Length of the second array, within the calldata , highlighted](https://static.wixstatic.com/media/706568_aa19bd23373f44a596439950d265fd51~mv2.png/v1/fill/w_740,h_344,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_aa19bd23373f44a596439950d265fd51~mv2.png)

### The address items in the second array

And that's how we convert the function `transfer([[123, 123], [123]], [0x5b38da6a701c568545dcfcb03fcb875f56beddc4,0x7b38da6a701c568545dcfcb03fcb875f56bedfb3])` to its Hexadecimal representation for the EVM.

![Address items in the second array, within the calldata, highlighted](https://static.wixstatic.com/media/706568_e1289b538b074fb4a2abbfc495e7badb~mv2.png/v1/fill/w_740,h_308,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_e1289b538b074fb4a2abbfc495e7badb~mv2.png)

The following video summarizes the calldata from this section's example:

<video src="https://video.wixstatic.com/video/706568_87da162471bf43169763c48738e89ee9/1080p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

## A triple nested array animation

Below we show a video demonstrating how a 3D uint array is ABI encoding: `f(uint[][][] memory data)`:

<video src="https://video.wixstatic.com/video/706568_8fa44b44606645e29888a23532238b74/720p/mp4/file.mp4" type="video/mp4" autoplay loop muted controls></video>

## Calldata length and gas cost

As a Solidity developer, one of your key concerns is [saving gas](https://www.rareskills.io/post/gas-optimization). More so, working with calldata comes with extra cost — every byte in calldata costs gas.   

To determine the cost of a call data we first need to figure out the length of the calldata by counting the bytes. Let's use our previous calldata string as a case study:

```solidity
0xa9059cbb0000000000000000000000003f5047bdb647dc39c88625e17bdbffee905a9f4400000000000000000000000000000000000000000000011c9a62d04ed0c80000
```

We'll first remove the `0x` as it's only a prefix for us to understand that it's an Ethereum related Hexadecimal. Now, we'll be left with this:

```solidity
a9059cbb0000000000000000000000003f5047bdb647dc39c88625e17bdbffee905a9f4400000000000000000000000000000000000000000000011c9a62d04ed0c80000
```

This string is 136 hex digits long, which represents 68 bytes. Each byte is represented by two characters (hex digits) in the calldata string. Therefore, we can calculate the length by dividing 136 by 2 = 68.

Each non-zero byte in [calldata costs 16 gas](https://eips.ethereum.org/EIPS/eip-2028), while zero bytes cost 4 gas. So, we need to separate them to proceed with out calculation.    
We have:   
- $32 \text{ non-zero bytes} = 32 \times 16 = 512 \text{ gas}$
- $36 \text{ zero bytes} = 36 \times 4 = 144 \text{ gas}$
    
Total gas cost for calldata $= 512 \text{ gas} + 144 \text{ gas} = 656 \text{ gas}$.

Since zero bytes are cheaper, some developers mine for addresses or smart contract addresses with several leading zero bytes as this reduces the gas cost for passing that address as an argument.

## Conclusion

Throughout this guide, we've learned the basics of ABI encoding for function calls, the key components of an ABI-encoded function call, and gained a more detailed understanding of calldata. We've also explored how to calculate the gas cost of calldata and even went further to explore more complex calldata decoding and encoding excercise to help solidify the knowledge, I hope you found it useful. To further enforce what you have learned in this article, I recommend reading more about the [Ethereum ABI encoding spec](https://docs.soliditylang.org/en/latest/abi-spec.html) and practicing the problems in the next section.

Happy  encoding!

## Practice Problems

1. How many bytes are in the calldata for a call to `foo(uint16 x)`?
2. What is the ABI encoding for `foo(uint256 x, uint256[])` when passed `(2, [5, 9])`?
3. What is the ABI encoding for `foo(S[] memory s)` where `S` is a struct with fields `uint256 x; uint256[] a;`? (Credit to this [tweet](https://x.com/z0age/status/1636048648009285639?s=46) for inspiration).
    
## Capture the Flag Exercises

RareSkills Solidity Riddles: [Forwarder](https://github.com/RareSkills/solidity-riddles)     
DamnVulnerableDeFi: [ABI Smuggling](https://www.damnvulnerabledefi.xyz/challenges/abi-smuggling/)

## Authorship

This article was written by [Eze Sunday](https://linkedin.com/in/ezesundayeze) in collaboration with RareSkills.

*Originally Published May 29*
