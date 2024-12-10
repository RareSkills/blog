# ERC-1155 Multi Token Standard

The ERC-1155 standard describes how to create both fungible and non-fungible tokens then incorporate them into a single smart contract. This saves significant deployment costs when several tokens are involved.

Imagine you are a game developer trying to incorporate NFTs and ERC-20 tokens into your platform, representing various types of assets like shoes, swords, hats, and in-game currency.

Using standards like ERC-721 and ERC-20 would require you to develop multiple token contracts, one for each collection of NFTs and ERC-20s. Deploying all those contracts would be expensive.

Wouldn’t it be convenient if you could define and manage all of the NFT assets and tokens within a single contract? Then, you could even create a mechanism to approve or transfer multiple NFTs at once.

This use-case is why Enjin, an NFT and Gaming development organization, submitted the first proposal of the ERC-1155 Multi Token Standard to Ethereum's Github repository. On the 17th of June 2018, Enjin's ERC-1155 token standard was officially adopted by the Ethereum Foundation.

## ERC-1155’s Key Features

### Handling Multiple Token Types: fungible and non-fungible

To account for multiple types of tokens (fungible and/or non-fungible) within a single contract, an ERC-1155 implementation must distinguish each token type using a unique `uint256` token ID. This allows contracts to define unique attributes for each token such as total supply, URI, name, symbol, etc. and ensures that each token’s configuration remains separate and independent from one another.

Here is an example of an ERC-1155 token ID structure:

- **Token ID: 0**
- **Token ID: 1**
- **Token ID: 2**
- …

Token IDs do not have to be sequential. They just have to be unique. The standard doesn’t state how token IDs should be created, thus **the “mint” function is not part of the specification.**

### Fungibility Definition

Here are the definitions of fungible and non-fungible tokens; ERC-1155 supports both.

- **Fungible**
    
    These are tokens that are identical to each other, like units of currency. To define a fungible token set in ERC-1155, you would simply mint multiple tokens for a given token ID.
    
    When each token shares the same id, they will also have the same name and symbol. This will allow the token to operate in the same way an ERC-20 does because it would have multiple units that are identical to each other, under the same name and symbol. Unlike ERC-20, there is no decimals with which to interpret fungible token quantities. All fungible token balances are presented in whole units.
    

- **Non-Fungible**
    
    Non-fungible tokens (NFTs) in ERC-1155 are unique tokens, each one distinct from the others. These are represented by assigning each unique item its own token ID, which is a unique `uint256` value.
    

## How to put multiple non-fungible tokens into a ERC-1155

When managing multiple NFT collections within a single ERC-1155 contract, assigning random unique token IDs can make it challenging to identify which collection a particular token ID belongs to. 

To address the issue, one solution is to structure the token IDs in a way the id encodes both the collection and individual item information: we simply concatenate the two numbers together, and the number formed by the concatenation is the id.

Here’s how it is done:

We divide the `uint256` token ID into two parts:

1. collection ID: the upper bits (the most significant 128 bits) of the token ID to represent a specific collection.
2. item ID: the lower bits (the least significant 128 bits) to represent an individual item within that collection.

This scheme enables us to easily identify which collection the token ID belongs to, and which item it is within that collection. All non-fungible tokens will be distinct from each other using this encoding.

The following image shows the token ID divided into the collection id (`X` values) and item ID (`Y` values):

![image shows the token ID divided into the collection id (`X` values) and item ID (`Y` values)](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/diagaram-256-bits-two-parts.png)

To encode collection and item information into a single `uint256` token ID, we can use bit-shifting and addition operation.

**Bit-Shifting**

Bit-shifting is the process of adding zero bits to the beginning or end of a bit sequence, essentially shifting the existing bits to the left (Solidity operation `<<`) or right (`>>`). 

By bit-shifting, we can “inject” a 128-bit number into the most significant 128 bits of the 256 bit number. By default, if we cast a 128-bit number to a 256-bit number, the 128 bit number would be at the least significant 128 bits.

Consider this 256-bit (or 32 bytes) value representing the decimal number `2`, shifted left by 128 bits (or 16 bytes):

<video src="https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/movie-bitshift-only.mp4" type="video/mp4" autoplay loop muted controls></video>

After shifting the decimal value `2` to the left by 128 bits (`2 << 128`), we get the new decimal value `680564733841876926926749214863536422912` or 0x0000000000000000000000000000000200000000000000000000000000000000

in hex.

Using this bit-shifting technique, we are able to pad the least significant 128 bits with zeroes. Since the NFT IDs are stored as `uint256` types, we can add the `item id` to the `shifted collection id`. Below is a simple formula to illustrate this:

`uint256 token_ID = shifted_collection_id + individual_token_id`

The following animation shows how the shifting and addition operations happen behind the scene:

<video src="https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/movie-bitshift-add.mp4" type="video/mp4" autoplay loop muted controls></video>

Imagine this scenario, an ERC-1155 contract has two different non-fungible token collections: CoolPhotos and RareSkills, with `collectionID` 1 and 2, respectively. If Bob wants to check if he owns an item with `itemID` 7 from the RareSkills collection, the valid token ID to use for this check would be a combination of the `collectionID` and `itemID`:

$\texttt{0x\textcolor{orange}{00000000000000000000000000000002}\textcolor{lightgreen}{00000000000000000000000000000007}}$

Where orange bits represents RareSkills collection ID and green bits represents the collection’s itemID.

Here’s how the ERC-1155 contract from the above example might store and retrieve account balances for a given token ID:

```solidity
// Nested mapping to store balances
// tokenID => owner => balance
mapping(uint256 => mapping(address => uint256)) balances;

// Retrieves the balance of a specific token ID for an address
function balanceOf(address owner, uint256 tokenid) public view returns (uint256) {
    return balances[tokenid][owner];
}
```

Using the above code, Bob could call the `balanceOf` function with the token ID `(2 << 128) + 7` to check his ownership:

```solidity
uint256 rareSkillsTokenCollectionID = 2 << 128; // collection id is 2
uint256 rsNFT = 7; // item id

// Returns 1 if Bob owns the tokenid passed, else, 0
uint256 bobBalance = balanceOf(
											address(Bob), 
										  rareSkillsTokenCollectionID + rsNFT  // (2 << 128) + 7 
										 );
```

If `bobBalance = 1`, Bob owns the item with `itemID` 7 from the RareSkills collection. It is critical that the contract enforce the total supply for this token cannot exceed 1, otherwise the token would become fungible instead of non-fungible.

We previously discussed using the bit-shifting method to uniquely compute token IDs. To reverse this process and get the `collectionId` and `itemId` from a `tokenId`, we shift the `tokenId` right by 128 bits to retrieve the `collectionId` and cast the `tokenId` to 128 bits to obtain the `itemId`.

Below is an example code on how to compute:

1. the ERC-1155 token ID given the NFT collection ID and itemId
2. the collection ID and item ID, given the ERC-1555 token ID

```solidity
contract A {

		// 1. COMPUTE TOKEN ID
    function getTokenId(
        uint256 collectionId, 
        uint256 itemId 
        ) public pure returns (bytes32 tokenId) {
        // shift the collection id by 128 to the left
        uint256 shiftedCollectionId = collectionId << 128;

        // add the item id to the shifted collection id
        tokenId =  bytes32(shiftedCollectionId + itemId);
    }

		// 2. GET COLLECTION ID AND ITEM ID
    function getCollectionIdAndItemId(
        uint256 tokenId
        ) public pure returns (uint256 collectionId, uint256 itemId) {
        // shift the token id to the right by 128 
        collectionId = tokenId >> 128;
        
        // cast the token id to 128
        itemId = uint128(tokenId);
    }
    
}
```

Screenshot from [Remix](https://remix.ethereum.org/?#code=Ly8gU1BEWC1MaWNlbnNlLUlkZW50aWZpZXI6IEdQTC0zLjAKcHJhZ21hIHNvbGlkaXR5IF4wLjguMjg7Cgpjb250cmFjdCBBIHsKCiAgICBmdW5jdGlvbiBnZXRUb2tlbklkKAogICAgICAgIHVpbnQyNTYgY29sbGVjdGlvbklkLCAKICAgICAgICB1aW50MjU2IGl0ZW1JZCAKICAgICAgICApIHB1YmxpYyBwdXJlIHJldHVybnMgKGJ5dGVzMzIgdG9rZW5JZCkgewogICAgICAgIC8vIHNoaWZ0IHRoZSBjb2xsZWN0aW9uIGlkIGJ5IDEyOCB0byB0aGUgbGVmdAogICAgICAgIHVpbnQyNTYgc2hpZnRlZENvbGxlY3Rpb25JZCA9IGNvbGxlY3Rpb25JZCA8PCAxMjg7CgogICAgICAgIC8vIGFkZCB0aGUgaXRlbSBpZCB0byB0aGUgc2hpZnRlZCBjb2xsZWN0aW9uIGlkCiAgICAgICAgdG9rZW5JZCA9ICBieXRlczMyKHNoaWZ0ZWRDb2xsZWN0aW9uSWQgKyBpdGVtSWQpOwogICAgfQoKICAgIGZ1bmN0aW9uIGdldENvbGxlY3Rpb25JZEFuZEl0ZW1JZCgKICAgICAgICB1aW50MjU2IHRva2VuSWQKICAgICAgICApIHB1YmxpYyBwdXJlIHJldHVybnMgKHVpbnQyNTYgY29sbGVjdGlvbklkLCB1aW50MjU2IGl0ZW1JZCkgewogICAgICAgIC8vIGNhc3QgdGhlIHRva2VuIGlkIHRvIDEyOCBiaXRzCiAgICAgICAgaXRlbUlkID0gdWludDEyOCh0b2tlbklkKTsKCiAgICAgICAgLy8gc2hpZnQgdGhlIHRva2VuIGlkIHRvIHRoZSByaWdodCBieSAxMjggCiAgICAgICAgY29sbGVjdGlvbklkID0gdG9rZW5JZCA+PiAxMjg7CiAgICB9Cgp9&lang=en&optimize=false&runs=200&evmVersion=null&version=soljson-v0.8.28+commit.7893614a.js) showing the code being tested for both functions:

![Remix code sceenshot calling both functions](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-remix-parse-id.png)

The structured token ID technique is one way to implement multiple non-fungible tokens with the ERC1155, since the standard does not specify how it must be done. However, there is an implementation of ERC1155 called [ERC1155D](https://medium.com/donkeverse/introducing-erc1155d-the-most-efficient-non-fungible-token-contract-in-existence-c1d0a62e30f1) which is an iteration on the original standard to optimize the gas efficiency for minting non-fungible tokens, if the contract only needs to support a single NFT collection.

### **ERC-1155D**

ERC-1155D is designed specifically for non-fungible tokens (same as ERC-721) where each token has a unique identifier and a unique owner. It is fully backwards compatible and compliant with ERC-1155.

**When to use ERC1155D?**

Use ERC1155D when you don’t need multiple non-fungible token collections (like with the CoolPhotos RareSkills example) in your contract , while also enforcing that token have a supply of one and a maximum of one owner. 

In summary, all tokens are managed under a single contract using a `uint256` value for the token ID. However, how specific token IDs are assigned to different types of tokens is entirely up to the contract’s use case.

## Core ERC1155 Functions

These are functions in the ERC1155 interface that must be implemented by contracts implementing the ERC1155 standard. The code snippet from each function is from the standard’s specification.

### Balance Retrieval

- **balanceOf**
    
    In ERC-721, `balanceOf(address _owner)` returns an address balance for the entire collection of token IDs. So if an address owns tokens 1, 5 and 7, `balanceOf(address _owner)` for that address would return `3`.
    
    However, in ERC-1155, the `balanceOf` function is structured in a way that it retrieves the token balance of a specific token ID for a particular account address.
    
    ```solidity
    /**
        @notice Get the balance of an account's tokens.
        @param _owner  The address of the token holder
        @param _id     ID of the token
        @return        The _owner's balance of the token type requested
    */
    function balanceOf(address _owner,uint256 _id) external view returns (uint256);
    ```
    
    An address can hold different amounts of various token ID, such as one unit of token ID 1, twenty units of token ID 5 and so forth. However, there is no direct way to measure the total number of tokens an address owns across all token IDs within an ERC-1155 contract, because the `balanceOf` function is designed to only check **how much of a particular tokenID you own** and not how many tokenIDs you own across the entire contract.
    
- **balanceOfBatch**
    
    There also exists a batch mechanism called `balanceOfBatch(address[] calldata _owners, uint256[] calldata _ids)`. This method retrieves multiple balances per address per id at once by calling `balanceOf` in a loop.
    
    ```solidity
    /**
        @notice Get the balance of multiple account/token pairs
        @param _owners The addresses of the token holders
        @param _ids    ID of the tokens
        @return        The _owner's balance of the token types requested (i.e. balance for each (owner, id) pair)
    */
    function balanceOfBatch(
    		address[] calldata _owners,
    		uint256[] calldata _ids
    		) external view returns (uint256[] memory);
    ```
    
    ERC-1155 does not support a mechanism to list all the existing token IDs.
    
    To get all the existing IDs of a 1155 contract, we must parse the logs off chain (we will show how to do this later).
    

### **Approval For All**

ERC-1155 allows an owner to grant an operator approval to manage all of their tokens across all IDs in a single transaction by calling its `setApprovalForAll(address _operator, bool _approved)` method. This function takes an operator’s `address` and a `bool` representing the approval status as parameters:

```solidity
/**
    @notice Enable or disable approval for a third party ("operator") to manage all of the caller's tokens.
    @dev MUST emit the ApprovalForAll event on success.
    @param _operator  Address to add to the set of authorized operators
    @param _approved  True if the operator is approved, false to revoke approval
    */
function setApprovalForAll(address _operator,bool _approved) external;
```

Be aware that this method approves literally everything the user owns in the ERC-1155 contract. It’s like setting maximum approval for ERC-20 and calling setApprovalForAll for ERC-721. Any of the owner’s token in any quantity in the ERC-1155 contract can be transferred by the operator.

### **Safe Transfers**

Following the ERC-721 pattern, ERC-1155 also features the [“safe transfer” mechanism](https://www.rareskills.io/post/erc721#viewer-7r7m6), which checks to ensure that the recipient of the token is a valid receiver. In fact, ERC-1155 ONLY supports safe transfers.

- **safeTransferFrom**
    
    ```solidity
    /**
        @param _from    Source address
        @param _to      Target address
        @param _id      ID of the token type
        @param _value   Transfer amount
        @param _data    Additional data with no specified format, MUST be sent unaltered in call to `onERC1155Received` on `_to`
    */
    function safeTransferFrom(
    			address _from,
    			address _to,
    			uint256 _id,
    			uint256 _value,
    			bytes calldata _data) external;
    ```
    
    If the recipient is an EOA, then `safeTransferFrom` checks that the address is not the zero address. If the recipient is a smart contract then it will call the `onERC1155Received(address _operator, address _from, uint256 _id, uint256 _value, bytes calldata _data)` callback function and expect the magic value `bytes4(keccak256("onERC1155Received(address,address,uint256,uint256,bytes)"))` to be returned.
    
    An ERC-1155 token cannot be transferred to a smart contract that does not implement `onERC1155Received` or implements `onERC1155Received` incorrectly.
    
- **safeBatchTransferFrom**
    
    ```solidity
    /**
        @param _from    Source address
        @param _to      Target address
        @param _ids     IDs of each token type (order and length must match _values array)
        @param _values  Transfer amounts per token type (order and length must match _ids array)
        @param _data    Additional data with no specified format, MUST be sent unaltered in call to the `ERC1155TokenReceiver` hook(s) on `_to`
    */
    function safeBatchTransferFrom(
    			address _from,
    			address _to,
    			uint256[] calldata _ids,
    			uint256[] calldata _values,
    			bytes calldata _data) external;
    ```
    
    Furthermore, the standard allows for owners and operators to execute batch transfers. Multiple sets of tokens can be transferred from a source address to a destination address in a single transaction.
    
    Batch transfers can be performed by calling:
    
    - `safeBatchTransferFrom(address _from, address _to, uint256[] calldata _ids, uint256[] calldata _values, bytes calldata _data)`,
        - which will call the `onERC1155BatchReceived(address _operator, address _from, uint256[] calldata _ids, uint256[] calldata _values, bytes calldata _data)` callback on the receiver
            - and expect the magic value `bytes4(keccak256("onERC1155BatchReceived(address,address,uint256[],uint256[],bytes)"))`.
    
    **SafeTransferFrom vs SafeBatchTransferFrom**
    
    Using the OpenZeppelin’s implementation of ERC1155, the image below compares the gas usage of calling `safeTransferFrom` three times versus batching the transfers into a single transaction:
    
    ![Gas benchmark of SafeTransferFrom and SafeBatchTransferFrom](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-gas-benchmark.png)
    

Using `safeBatchTransferFrom`, as seen in the red box, consumes 132,437 gas, which is significantly lower than the 189,861 gas used for three separate `safeTransferFrom` calls, shown in the blue box. 

## Core Data Structures

ERC-1155 implementations usually uses mappings to hold the state of the core data such as the aforementioned balances, approvals and URIs. For example, an ERC-1155 could use the following storage variables:

```solidity
mapping(uint256 id => mapping(address account => uint256 balance)) internal _balances;

mapping(address account => mapping(address operator => bool isApproved)) internal _operatorApprovals;

string private _uri;
```

Let’s examine each of these data structures in the following sections.

### Balances

Balances are stored in a nested mapping with two levels. The outer mapping has a key that represents a `token ID` that points to another mapping of `address` (owner) to `_balances`.

To return an account’s balance for a given token under this structure, a `balanceOf` implementation would access the value as such:

```solidity
function balanceOf(address account, uint256 id) public view returns (uint256) {
    return _balances[id][account];
}
```

### Approvals

Similarly, approvals are held in a nested mapping since an account can grant approvals to multiple operators. The key of the outer mapping is the owner that points to a mapping of operators to their approval status.

Consider this example implementation of the `isApprovedForAll` function accessing an approval status:

```solidity
function isApprovedForAll(address account, address operator) public view returns (bool) {
    return _operatorApprovals[account][operator];
}
```

### Logging and Events

The ERC-1155 standard guarantees that an accurate record of all current token balances can be created by observing the event logs that were emitted by the smart contract because every token mint, burn, and transfer is logged.

A list of the scenarios that must emit an event is as follows:

- When an address grants or revokes operator approval for another address to manage all their tokens, the `ApprovalForAll` event must be emitted:
    
    ```solidity
    event ApprovalForAll(address indexed _owner, address indexed _operator, bool _approved);
    ```
    
- When tokens are transferred from one address to another, including minting and burning, the `TransferSingle` or `TransferBatch` event must be emitted.
    
    ```solidity
    // Emits when a single token is transferred
    event TransferSingle(address indexed _operator, address indexed _from, address indexed _to, uint256 _id, uint256 _value);
    
    // Emits when a batch of tokens is transferred
    event TransferBatch(address indexed _operator, address indexed _from, address indexed _to, uint256[] _ids, uint256[] _values);
    ```
    
    If the `safeBatchTransferFrom` function is called with a single tokenID, the `TransferSingle` event is emitted, else, the `TransferBatch` event is emitted.
    
- If the metadata URI changes for a specific token ID, the `URI` event must be emitted:
    
    ```solidity
    event URI(string _value, uint256 indexed _id);
    ```
    

With these events logged/emitted anytime the function associated with them is called, we can get information such as the following off-chain in JavaScript:

- **Existing token IDs of a 1155 contract:**
    
    The code below uses ethers.js library to interact with an ERC-1155 contract and fetch a list of all token IDs emitted in `TransferSingle` and `TransferBatch` events within a specified block range.
    
    ```jsx
    import { ethers } from "ethers"; // v6
    
    // Connect to an Ethereum provider
    const provider = new ethers.JsonRpcProvider("rpc-url");
    
    // ERC-1155 contract address and ABI
    const erc1155ContractAddress = "YourContractAddress";
    const abi = [
      /* ERC-1155 ABI here */
      "event TransferSingle(address indexed _operator, address indexed _from, address indexed _to, uint256 _id, uint256 _value)",
      "event TransferBatch(address indexed _operator, address indexed _from, address indexed _to, uint256[] _ids, uint256[] _values)",
    ];
    const contract = new ethers.Contract(erc1155ContractAddress, abi, provider);
    
    (async (startBlockNumber) => {
      // Fetch `TransferSingle` and `TransferBatch` events
      const singleEvents = await erc1155ContractInstance.queryFilter(
        "TransferSingle", // event
        startBlockNumber, // start block
        startBlockNumber + 100000, // end block
      );
      const batchEvents = await erc1155ContractInstance.queryFilter(
        "TransferBatch", // event
        startBlockNumber, // start block
        startBlockNumber + 100000, // end block
      );
    
      const tokenIds = new Set();
    
      // Get token IDs from TransferSingle events
      singleEvents.forEach((event) => {
    	  // Destructure the `args` field
        const { operator, from, to, id, value } = event.args;
        
        // Add `id` to the `tokenIds` set
        tokenIds.add(id);
      });
    
      // Get token IDs from TransferBatch events
      batchEvents.forEach((event) => {
      	// Destructure the `args` field
        const { operator, from, to, ids, values } = event.args;
        
        // Loop through `ids` then add `id` to the `tokenIds` set
        ids.forEach((id) => tokenIds.add(id.toString()));
      });
    
      console.log("Token IDs in existence:", Array.from(tokenIds));
    })();
    ```
    
- **All the token IDs a user owns:**
    
    The code below lists all the IDs owned by a user. It does this by tracking transfer events `to` and `from` that address. For accuracy, the `startBlockNumber` needs to be set before the earliest interaction of that address.
    
    ```jsx
    async function getUserTokenIds(userAddress, startBlockNumber) {
        const singleEvents = await erc1155ContractInstance.queryFilter('TransferSingle', startBlockNumber, startBlockNumber + 100000);
        const batchEvents = await erc1155ContractInstance.queryFilter('TransferBatch', startBlockNumber, startBlockNumber + 100000);
    
        const balances = {};
    
        // Process TransferSingle events
        singleEvents.forEach(event => {
            const { operator, from, to, id, value } = event.args;
    
            if (to.toLowerCase() === userAddress.toLowerCase()) {
                balances[id] = (balances[id] || 0) + parseInt(value.toString());
            }
    
            if (from.toLowerCase() === userAddress.toLowerCase()) {
                balances[id] = (balances[id] || 0) - parseInt(value.toString());
            }
        });
    
        // Process TransferBatch events
        batchEvents.forEach(event => {
            const { operator, from, to, ids, values } = event.args;
    
            ids.forEach((id, index) => {
                const value = parseInt(values[index].toString());
    
                if (to.toLowerCase() === userAddress.toLowerCase()) {
                    balances[id] = (balances[id] || 0) + value;
                }
    
                if (from.toLowerCase() === userAddress.toLowerCase()) {
                    balances[id] = (balances[id] || 0) - value;
                }
            });
        });
    
        // Filter out IDs with a balance greater than zero
        const ownedTokenIds = Object.keys(balances).filter(id => balances[id] > 0);
    
        console.log(ownedTokenIds);
    }
    ```
    

### Uniform Resource Identifiers (URIs)

ERC-1155 has only one `uri` function, as specified in the standard. The standard does not state whether the `uri` function should use or ignore the token ID. How the uri is retrieved depends on the contract's implementation. For example, If the contract implementation requires a shared URI, we can ignore the id and just return the base uri `_uri`, else, we can encode both the token ID and the base uri.

**An example implementation of Shared URI for token IDs:**

```solidity
string private _uri;

function uri(uint256 /* id */) public view virtual returns (string memory) {
    return _uri;
}
```

The above `uri` function will always return the same URI, ignoring the token ID. 

**An example implementation of Unique URI for each token ID:**

If we want to change the string returned based on the token ID, the `Strings` library will be handy, but it is not native to Solidity but is part of the [OpenZeppelin Strings library](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/Strings.sol). In the example implementation below, it is used to convert a `tokenID` which is a uint256, into a hexadecimal number encoded as a Solidity string. 

Below is an example of how to change the URI based on the ID using the `Strings` library:

```solidity
import "@openzeppelin/contracts/utils/Strings.sol";

string private _uri;

function uri(uint256 id) public view virtual returns (string memory) {
    return string(abi.encodePacked(
            _uri,
            Strings.toHexString(id, 32), // Convert tokenId to hex with fixed length
            ".json"
    ));
}
```

The `uri` function returns a unique URI for each token by appending the passed token ID to the base URI. For example, if the base URI is `https://token-cdn-domain/`, calling the function with token ID `314592` (in hex, 0x4CCE0) will return `https://token-cdn-domain/000000000000000000000000000000000000000000000000000000000004cce0.json`.

The standard requires that clients replace the `{id}` parameter (if present) with the hexadecimal string representation of actual id of the token. The substituted string must be lowercase alphanumeric: `[0-9a-f]` with no “0x” prefix and be leading zero padded to 64 hex characters length if necessary.

The token ID substitution approach reduces the overhead required to store unique URIs for large collections of tokens by appending the passed token ID to the base uri.

**How URIs Are Structured**

The standard does not require that ERC-1155 tokens must have URI metadata associated with them. However, if ERC-1155 implementation contracts do define any token’s URI, it must point to a JSON file that conforms to the “ERC-1155 Metadata URI JSON Schema”. 

This URI typically points to an off-chain resource, such as a server or IPFS, where the metadata is stored.

The ERC-1155 Metadata URI JSON Schema, as copied from the standard is outlined below:

```json
{
    "title": "Token Metadata",
    "type": "object",
    "properties": { 
        "name": {
            "type": "string",
            "description": "Identifies the asset to which this token represents"
        },
        "decimals": {
            "type": "integer",
            "description": "The number of decimal places that the token amount should display - e.g. 18, means to divide the token amount by 1000000000000000000 to get its user representation."
        },
        "description": {
            "type": "string",
            "description": "Describes the asset to which this token represents"
        },
        "image": {
            "type": "string",
            "description": "A URI pointing to a resource with mime type image/* representing the asset to which this token represents. Consider making any images at a width between 320 and 1080 pixels and aspect ratio between 1.91:1 and 4:5 inclusive."
        },
        "properties": {
            "type": "object",
            "description": "Arbitrary properties. Values may be strings, numbers, object or arrays."
        }
    }
}
```

An example JSON for a car NFT that aligns with the JSON metadata schema above:

```json
{
  "title": "RareSkills Car Metadata",
  "type": "object",
  "properties": {
    "name": "RareSkills Car #1",
    "description": "A high-performance electric car with cutting-edge technology.",
    "image": "https://image-uri/rareskills-car1.png",
    "year": 2024,
    "topSpeed": "200 mph",
    "batteryCapacity": "100 kWh",
    "features": ["Autopilot", "Full Self-Driving", "Premium Sound System"],
  }
}

```

The `title` field describes the purpose of the metadata, `type` field specifies the data format for the metadata, `properties` field defines additional attributes or metadata about the car.

### Localization field in the URI JSON Schema

Clients that support localization may be able to display token information in multiple languages by utilizing the `localization` attribute in the JSON formatted ERC-1155, if it exists.

The schema for the `localization` metadata is as follows:

```json
{
    "title": "Token Metadata",
    "type": "object",
    "properties": {
		    
		    ...
		    
        "localization": {
            "type": "object",
            "required": ["uri", "default", "locales"],
            "properties": {
                "uri": {
                    "type": "string",
                    "description": "The URI pattern to fetch localized data from. This URI should contain the substring `{locale}` which will be replaced with the appropriate locale value before sending the request."
                },
                "default": {
                    "type": "string",
                    "description": "The locale of the default data within the base JSON"
                },
                "locales": {
                    "type": "array",
                    "description": "The list of locales for which data is available. These locales should conform to those defined in the Unicode Common Locale Data Repository (http://cldr.unicode.org/)."
                }
            }
        }
    }
}
```

Below is an example of metadata JSON file that contains a `localization` attribute:

```json
{
 "name": "RareSkills Token",
 "description": "Each token represents a unique pass in RareSkills community.",
 "properties": {
	 "localization": {
	   "uri": "ipfs://xxx/{locale}.json",
	   "default": "en",
	   "locales": ["en", "es", "fr"]
	 }
 }
}
```

The `locales` property is an array with three elements: `en`, `es`, and `fr`, with `en` set as the default language. Each element in the array has its own metadata JSON file in its respective language.

es.json:

```json
{
  "name": "RareSkills simbólico",
  "description": "Cada token representa un pase único en la comunidad RareSkills."
}
```

fr.json:

```json
{
  "name": "RareSkills Jeton",
  "description": "Chaque jeton représente un pass unique dans la communauté RareSkills."
}
```

Similar to token ID substitution, if the `uri` contains the string `{locale}` then clients must replace this with one of the available locales which are defined in `locales` array which then points to a metadata JSON file in the target language.

**Example Steps to Get French Language Metadata**

1. Call the `uri` function with the token ID `314592` to get the URI for the token metadata JSON
    
    ```solidity
    // Returned uri: https://token-cdn-domain/000000000000000000000000000000000000000000000000000000000004cce0.json
    ```
    
2. Read the JSON content off-chain from the returned uri in step 1 to get the base URI for our desired language
    
    ```json
    {
     "name": "RareSkills Token",
     "description": "Each token represents a unique pass in RareSkills community.",
     "properties": {
    	 "localization": {
    	   "uri": "https://token-cdn-domain/000000000000000000000000000000000000000000000000000000000004cce0/{locale}.json",
    	   "default": "en",
    	   "locales": ["en", "es", "fr"]
    	 }
     }
    }
    ```
    
3. Replace the `{locale}` string with `fr` in `localization → uri` field to get the URI for the French version of the metadata
    
    ```solidity
    // French Language URI: 
    // [https://token-cdn-domain/000000000000000000000000000000000000000000000000000000000004cce0/fr.json](https://token-cdn-domain/000000000000000000000000000000000000000000000000000000000004cce0.json)
    ```
    

When interacting with an untrusted metadata, be sure to sanitize the results before parsing it. Any JSON rendered on a front end may be vector for cross site scripting attacks.

### How OpenSea interprets the metadata

ERC-1155 contracts are supported by OpenSea and this section shows how OpenSea interprets the ERC-1155 metadata. A live example is from a blockchain game called [Common Ground World](https://opensea.io/collection/commongroundworld):

![Screenshot of common worlds nft on opensea](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-opensea-common-worlds.png)

As at the time of writing, Common Ground World has 681 collections defined as in-game assets, referred to by OpenSea as “Unique items” (red box) in the image above. The sum of all assets in each collection is around `9 million` (green box).

Here is an example of one of the game’s collections:

![Screenshot of one of common worlds collection](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-opensea-watertank.png)

The [Water Tank](https://opensea.io/assets/ethereum/0xc36cf0cfcb5d905b8b513860db0cfe63f6cf9f5c/232753138973921909008948231483329456635904) collection has a total supply of about 4,800 items (green box) owned by about 2,900 addresses (red box).

Notice, OpenSea does not provide the total supply information for any given ERC-721 token because each tokenID has a supply of one and exactly one owner. Here is a random Bored Ape Yacht Club NFT owned by `F15C93`, for comparison:

![Screenshot of a random Bored Ape Yacht Club NFT owned by F15C93](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-opensea-bored-ape.png)

It’s more clear that this token is adhering to the ERC-1155 standard by observing the Details section from OpenSea, see the red box below:

![Screenshot of common world metadata](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-opensea-metadata.png)

OpenSea is able to display the description and trait information by pulling from the token’s metadata, which can be observed by clicking on the [Token ID](https://tokens.gala.games/metadata/0xc36cf0cfcb5d905b8b513860db0cfe63f6cf9f5c/232753138973921909008948231483329456635904):

```json
{
    "decimalPlaces": 0,
    "description": "Never underestimate the power of passive, on-demand water for your crops. Your Farmers will thank you!",
    "image": "https://tokens.gala.games/images/sandbox-games/town-star/storage/water-tank.gif",
    "name": "Water Tank",
    "properties": {
        "category": "Storage",
        "game": "Town Star",
        "rarity": {
            "hexcode": "#939393",
            "icon": "https://tokens.gala.games/images/sandbox-games/rarity/common.png",
            "label": "Common",
            "supplyLimit": 5159
        },
        "tokenRun": "storage"
    }
}
```

OpenSea defines [metadata standards](https://docs.opensea.io/docs/metadata-standards) that URIs must adhere to in order for OpenSea to pull in off-chain metadata for ERC721 and ERC1155 assets.

## Implementation Example

The following is an example ERC-1155 implementation contract which is a simple game. It instantiates [OpenZeppelin’s ERC-1155 abstract contract](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC1155/ERC1155.sol) with additional functions serving as wrappers and helpers to alter the game state:

- `initializePlayer`: initializes a player's account by minting an amount defined by the constant `INITIAL_IN_GAME_CURRENCY_BALANCE`.
- `mintInGameCurrency`: mints additional in-game currency for a specific player.
- `mintCar`:  allows players to mint unique NFT-based cars.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import { ERC1155 } from "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";

contract GameAssets is ERC1155 {
    uint256 constant TOKEN_ID_IN_GAME_CURRENCY = 0;    // fungible tokenId 
    uint256 constant TOKEN_ID_BASE_CAR_COLLECTION = 1; // non-fungible tokenId 
    
    uint256 constant INITIAL_IN_GAME_CURRENCY_BALANCE = 1000;
    uint256 constant MINIMUM_AMOUNT = 1500;

    uint256 public nextTokenIndex;

    constructor(string memory uri) ERC1155(uri) {}

    function initializePlayer(address to, bytes memory data) public {
				mintInGameCurrency(to, INITIAL_IN_GAME_CURRENCY_BALANCE, data);
    }

    function mintInGameCurrency(address to, uint256 value, bytes memory data) public {
        _mint(to, TOKEN_ID_IN_GAME_CURRENCY, value, data);
    }

    function mintCar(address player, bytes memory data) public returns (uint256 carId) {
		    // ASSERT PLAYER'S BALANCE OF THE TOKEN ID `TOKEN_ID_IN_GAME_CURRENCY`
		    // EQUALS OR GREATER THAN `MINIMUM_AMOUNT`
		    require(balanceOf(player, TOKEN_ID_IN_GAME_CURRENCY) >= MINIMUM_AMOUNT, "");
		    
        // THE NON-FUNGIBLE MAGIC TRICK
        carId = (TOKEN_ID_BASE_CAR_COLLECTION << 128) + nextTokenIndex++;

				// MINT CAR
        _mint(player, carId, 1, data);
    }
}
```

**NOTE:** This contract is only for demonstration purposes and omits key security features and optimizations.

The game is going to have two types of tokens:

1. An in-game currency ($IGC) that a player can earn by completing quests. This is going to be a fungible token.
2. A non-fungible token that represents a collection of cars that players can mint.

When we deploy this contract, our contract address is `0xCc3958FE4Beb3bcb894c184362486eBEc2E1fD4D` and we will be using `0x5B38Da6a701c568545dCfcB03FcB875f56beddC4` as the player address.

In the next few sections we will demonstrate how to interact with this contract to manage its token assets.

## A game example using ERC-1155

The flowchart below illustrates how players interact with the game’s ERC-1155 contract, including minting In-Game currency and cars.

![The game contract workflow diagram](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/game-workflow-diagram.png)

### ERC-1155 Token 0: Mint $IGC

![IGC token image](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/IGC.jpeg)

Let’s say we want players to start with a balance of `1000` $IGC. We can mint these tokens to each player at the start of the game by calling the `initializePlayer` function in our contract. This will send the token ID for IGC (`0`) and the amounts to mint to `_mint(address to, uint256 id, uint256 value, bytes memory data)` of the OpenZeppelin base contract.

This `_mint` function is OpenZeppelin’s method for creating tokens and it eventually performs the acceptance checks, calls `safeTransferFrom`and emits the `TransferSingle` event (blue box below), as required by the standard.

After calling the `initializePlayer` function we can see the following logs:

![Screenshot showing the transfer single event](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-transfer-single-event.png)

In the red box, we can see that the `TransferSingle` event was emitted, and in the green box the zero address sent `1000` units of our in-game currency (token ID `0`) to our player’s address. 

### Mint more $IGC

As our player completes quests, we want to reward them with more $IGC. We can call the `mintInGameCurrency` function in the game contract, which then calls OpenZeppelin’s `_mint` function, specifying our player’s address (`0x5B38Da6a701c568545dCfcB03FcB875f56beddC4`), the amount of tokens to mint as a reward (`500`) and any byte data to send to the the receiver’s callback (no data in this case). Calling `mintInGameCurrency` with these values will mint 500 $IGC tokens to the target address, giving a total balance of 1500 $IGC tokens.

When we check our player’s $IGC balance via `balanceOf`:

![Screenshot of the player’s $IGC balance](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-game-balance.png)

We see that our player now has a balance of `1500` $IGC (initial + reward).

### ERC-1155 Token 1: Minting Non-Fungible Assets (Cars)

![car nft](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/car.jpg)

Now, let’s say we want to allow players to mint cars by having a minimum $IGC balance. Keep in mind, the car collection is non-fungible.

First, we’ll define unique metadata for each car NFT that contains the car’s features.

For example, the URI first car in our collection will be:

`https://token-cdn-domain/0000000000000000000000000000000100000000000000000000000000000000.json` 

where id is:

$\texttt{340282366920938463463374607431768211456}$ in decimal

or

$\texttt{0x\textcolor{orange}{00000000000000000000000000000001}\textcolor{lightgreen}{00000000000000000000000000000000}}$ in hex

The orange bits represent the car collection ID (`1`), while the green bits represent the first car token ID (`0`). Together, they form a unique id that points to a metadata, let’s say:

```json
{
    "name": "Super Fast Car",
    "description": "This super fast car is not like any other, it's super fast.",
    "image": "https://images.com/{id}.png",
    "properties": {
        "features": {
            "speed": "100",
            "color": "blue",
            "model": "SuperFast x1000",
            "rims": "aluminum"
        }
    }
}
```

Now, we call `mintCar` function on our contract to mint their car NFT:

```solidity
function mintCar(address player, bytes memory data) public returns (uint256 carId) {
		// ASSERT PLAYER'S BALANCE OF THE TOKEN ID `TOKEN_ID_IN_GAME_CURRENCY`
    // EQUALS OR GREATER THAN `MINIMUM_AMOUNT`
		require(balanceOf(player, TOKEN_ID_IN_GAME_CURRENCY) >= MINIMUM_AMOUNT, "");
		    
    // THE NON-FUNGIBLE MAGIC TRICK
    carId = (TOKEN_ID_BASE_CAR_COLLECTION << 128) + nextTokenIndex++;

	  // MINT CAR
    _mint(player, carId, 1, data);
}
```

The `carId` variable is where the non-fungible magic happens. It calculates a unique token ID for each car NFT by combining the car collection ID and the next available token index (starts from zero).

After calling `mintCar` function:

![Screenshot of the log after the mintCar function is called](https://pub-32882f615aa84e4a94e1279ccf3ab85a.r2.dev/ERC-1155-media/screenshot-log-from-to.png)

As expected, minted a single car NFT (yellow box) to the player’s address from `address zero`.

**NOTE:** The ID of the NFT (red box) is `340282366920938463463374607431768211456` , which is the result of `(1 << 128) + 0` , with `1` being the base token ID for the car collection and `0` being the itemID of the NFT within the collection.

Beyond managing both fungible and non-fungible tokens within a single contract, it’s also important to address security vulnerabilities in ERC-1155 contracts. One common vulnerability is reentrancy attacks, which can exploit the minting or transferring process.

## Reentrancy Attacks In Minting and Transferring In ERC-1155

Due to the callback functions that are performed on `safeTransferFrom` and `safeBatchTransferFrom` operations, contracts using ERC-1155 are susceptible to re-entrancy attacks. ERC-1155 itself is safe, but adding code to it like an unsafe mint could introduce reentrancy.

Consider [this](https://github.com/RareSkills/solidity-riddles/blob/main/contracts/Overmint1-ERC1155.sol) contract from the [Solidity Riddles by RareSkills](https://github.com/RareSkills/solidity-riddles/tree/main) CTF challenges:

```solidity
// SPDX-License-Identifier: GPL-3.0
pragma solidity 0.8.15;
import "@openzeppelin/contracts/utils/Address.sol";
import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/token/ERC1155/utils/ERC1155Holder.sol";

contract Overmint1_ERC1155 is ERC1155 {
    using Address for address;
    mapping(address => mapping(uint256 => uint256)) public amountMinted;
    mapping(uint256 => uint256) public totalSupply;

    constructor() ERC1155("Overmint1_ERC1155") {}

    function mint(uint256 id, bytes calldata data) external {
        require(amountMinted[msg.sender][id] <= 3, "max 3 NFTs");
        totalSupply[id]++;
        _mint(msg.sender, id, 1, data);
        amountMinted[msg.sender][id]++;
    }

    function success(address _attacker, uint256 id) external view returns (bool) {
        return balanceOf(_attacker, id) == 5;
    }
}
```

Notice that the `mint` function tries to prevent the `msg.sender` from minting more than `3` NFTs. However, it does not contain a reentrancy lock nor does it operations follow the checks-effects-interactions pattern, since it checks the amount `msg.sender` has minted after it mints and performs the callback. Thus, an attacker could exploit this contract by calling the `mint` function from within their malicious contract’s `onERC1155Received` callback function, as the following exploit contract demonstrates:

```solidity
contract AttackOvermint1_ERC1155 {
    Overmint1_ERC1155 overmint1_ERC1155;

    constructor(Overmint1_ERC1155 _overmint1_ERC1155) {
        overmint1_ERC1155 = _overmint1_ERC1155;
    }

    function attackMint(uint256 id) external {
        overmint1_ERC1155.mint(id, "");
    }

    function onERC1155Received(address _operator, address _from, uint256 _id, uint256 _amount, bytes calldata _data) public returns (bytes4) {
        uint256 balance = overmint1_ERC1155.balanceOf(address(this), _id);

        if (balance < 5) {
            overmint1_ERC1155.mint(1, "");
        }

        return bytes4(keccak256("onERC1155Received(address,address,uint256,uint256,bytes)"));
    }
}
```

The attacker would first call a function in their malicious contract to initiate a mint. This will cause `msg.sender` to be the attacker’s contract. When the NFT is minted, `onERC1155Received` will be called on the attacker’s contract. This function checks to see if the desired amount has already been minted and if not then it reenters `mint` function.

It is important for ERC-1155 implementation contracts to mitigate against this vulnerability by strictly adhering to the [checks-effects-interactions pattern and/or implementing reentrancy locks](https://www.rareskills.io/post/where-to-find-solidity-reentrancy-attacks).

## Conclusion

ERC-1155 has standardized an interface for implementing multiple types of tokens within a single contract. This allows for gas saving mechanisms like batch operations and approvals for multiple tokens at once, as well as on deploying token contracts.

This standard eliminates the need to interact with multiple contracts when managing various token sets, improving gas efficiency and UX for blockchain games and other projects that use multiple tokens.