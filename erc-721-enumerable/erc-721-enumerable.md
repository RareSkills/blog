# How ERC721 Enumerable Works

An Enumerable ERC721 is an ERC721 with added functionality that enables a smart contract to list all the NFTs an address owns. This article describes how `ERC721Enumerable` functions and how we can integrate it into an existing ERC721 project. We'll use Open Zeppelin's popular implementation of [ERC721Enumerable](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/extensions/ERC721Enumerable.sol) for our explanation.

## Prerequisites

Since `ERC721Enumerable` is an extension of ERC721 , this article assumes that the reader has read our [ERC721 article](https://www.rareskills.io/post/erc721) or has knowledge about the ERC721 standard.

### Swap and Pop

Removing an item from an list in Solidity is typically done by copying the last element to the destination of the item that will be removed, then popping the array (deleting the last element). It's too expensive gas-wise to shift all the elements to the left. The operation to delete from a list is shown in the animation below, which removes the item at index 1 (number 5):

<video src="https://video.wixstatic.com/video/935a00_0e88f0ef81d54484b6c0c2f58f16da3c/720p/mp4/file.mp4" style="width: 100%; height: 100%;" autoplay loop muted controls></video>

## Why ERC721Enumerable?

To understand why we need an extension like `ERC721Enumerable`, let's consider an example scenario. If we had to find all the `NFTs` a wallet owns from a particular ERC721 contract, how would we do it with the functionality available within ERC721?

We would have to call the `balanceOf()` function with the token owner's address, which would give us the number of `NFTs` owned by that address. Then, we would loop over all the `tokenIDs` in the ERC721 contract and call the `ownerOf()` function for each of these `tokenIDs`.

Let's assume that the total supply of NFTs is 1000 and an address owns two NFTs, the first and the last. That is, it owns the `tokenIDs` #1 and #1000.

![An array of token ids](https://static.wixstatic.com/media/706568_dd37b28a663646c09792429ecc60a6cb~mv2.png/v1/fill/w_740,h_231,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_dd37b28a663646c09792429ecc60a6cb~mv2.png)

To find the 2 `tokenIDs` owned by the address (token #1 and token #1000, we would have to loop over all the `NFTs` in a contract and query `ownerOf()` on that `ID` (from 1 to 1000), which is computationally expensive. Furthermore, we don't always know all the tokenIDs in the contract, so we might not be able to do this.

In the upcoming sections, we'll learn how `ERC721Enumerable` solves this problem.

## Naïve Solution To Tracking Token Ownership

The naïve solution to tracking each token owned by an address is to store a mapping from the address to a list of owned NFTs.

```solidity
mapping(address owner => uint256[] ownedIDs) public ownedTokens;
```

However, this solution is inefficient and incomplete for the following reasons:

1.  If the user owns a lot of tokens, a smart contract reading their array might run out of gas storing the very long array in memory.
    
2.  There are more gas-efficient ways to store a list of data (discussed later).
    
3.  If we want to remove a particular token from the user's list of tokens, we need to scan the entire list to find it. If the array is very long, we might run out of gas.

To solve issues **1** and **2** ERC721 Enumerable uses an array instead of a mapping (see the next section) and to solve the 3rd issue, an additional data structure is needed, which maps the a `tokenID` to the index it is in.

### Using a mapping as an array

Mappings can be used in a manner similar to that of an array, where the keys are the _index_ and _values_ are the value stored at that index in the array.

![A diagram showing how a mapping can be used as an array](https://static.wixstatic.com/media/706568_e5cd098c77f4426b8e13fa38f77d13af~mv2.png/v1/fill/w_740,h_233,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_e5cd098c77f4426b8e13fa38f77d13af~mv2.png)

If we replace the array in our example above with a mapping, the `indexes` of the array become the key, and the `tokenIDs` become the values.

In Solidity, mappings are more gas efficient than arrays. The length of an array is implicitly checked whenever the array is indexed (i.e., for index `i`, it checks if `i < array.length`). This check increases the gas cost of using an array. Using a mapping as an array, we skip this check, and thus save gas.

However, unlike arrays, mappings don't have a built-in length property, which we could use to track the total number of `NFTs` in a contract. Therefore, mappings are not always a good substitute for arrays.

In the next section, we'll delve into each data structure from `ERC721Enumerable` individually.

## ERC721Enumerable: The Data Structures

ERC721 Enumerable tracks two things:

1.  all the `tokenIDs` in existence.
2.  all the `tokenIDs` an address owns.

To accomplish **1**, it uses the data structures `_allTokens` _and_ `_allTokensIndex`. 

To accomplish **2**, it uses the data structures `_ownedTokens` and `_ownedTokensIndex`

![The state variables of ERC-721 Enumerable highlighted](https://static.wixstatic.com/media/706568_8e2d616be798471a868f2afb394bd9db~mv2.png/v1/fill/w_740,h_170,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_8e2d616be798471a868f2afb394bd9db~mv2.png)

For the sake of simplicity, we'll use the same set of tokenIDs for every example and explanation i.e. _2, 5, 9, 7, and 1_.

###  <span style="color:Green">_allTokens array:</span>

![_allTokens array](https://static.wixstatic.com/media/706568_b5bcaa466ef74bef99277d1610f0652d~mv2.png/v1/fill/w_740,h_178,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_b5bcaa466ef74bef99277d1610f0652d~mv2.png)

The `_allTokens` array allows us to sequentially iterate over all `NFTs` in a contract. The `_allTokens` private array holds every existing `tokenID` (irrespective of its ownership status).

Initially, the order of `tokenIDs` in `_allTokens` depends on when they were minted. In the above diagram, `tokenID` `#2` is at index `#0` since it was minted before the other `tokenIDs`. This order can change upon burning of `tokenIDs`.

### <span style="color:#008aff">_allTokensIndex mapping</span>:

The `_allTokensIndex` mapping, given a tokenID, returns the index of that tokenID in the `_allTokens` array.

Instead of looping over `_allTokens` to find the index for a `tokenID`, we can use the `tokenID` itself to find its index in `_allTokens` using the `_allTokensIndex` mapping.

Being able to quickly find the `tokenID` enables the burn function to remove the `tokenID` efficiently.

![A diagram showing how _allTokensIndex holds the indexes of tokenIDs from _allTokens array](https://static.wixstatic.com/media/706568_cc9eb6a8736a4532978a8183cb047f9a~mv2.png/v1/fill/w_740,h_363,al_c,q_85,enc_auto/706568_cc9eb6a8736a4532978a8183cb047f9a~mv2.png)

The diagram above illustrates a mapping of `tokenIDs` to their corresponding index values. The `tokenID` #2 maps to the `0th` index since it was the first token minted in the contract. This mapping pattern continues for every token that gets minted.

### <span style="color:Red">_ownedTokens mapping:</span>

The _ownedTokens mapping is used to track the `tokenIDs` owned by an address. It has a nested mapping (i.e., `owner` -> `index` -> `tokenID`). It maps each `owner` address to an `index`, which is within range of the token balance of the address. Each index maps to a `tokenID` owned by that address.

![A diagram showing how the _ownedTokens mapping maps an address to index to tokenID](https://static.wixstatic.com/media/706568_5d6f6d4a377f48b488b7521e7cf503ea~mv2.png/v1/fill/w_740,h_304,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_5d6f6d4a377f48b488b7521e7cf503ea~mv2.png)

In the above diagram, the address '0xAli3c3' owns 3 NFTs, and thus has a mapping for 3 `tokenIDs`. The other address (0xb0b) owns a single token, and thus has a mapping for a single `tokenID`. At the index of #2, the nested mapping for the '0xAli3c3' address maps to the `tokenID` #1.

### <span style="color:#8d28a4">_ownedTokensIndex mapping:</span>

Just like how `_allTokensIndex` is the mirror image of `_allTokens`, `_ownedTokenIndex` is the mirror image of `_ownedTokens`.

`_ownedTokensIndex` is a mapping from tokenIDs to the index of that token in `_ownedTokens`, for that user. Consider the diagram below:

![A diagram showing how _ownedTokenIndex holds the index of a token in _ownedTokens](https://static.wixstatic.com/media/706568_d03e7a763a2a49949e48f0343398d314~mv2.png/v1/fill/w_740,h_279,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_d03e7a763a2a49949e48f0343398d314~mv2.png)

If we plug tokenID `2` or `9` into `_ownedTokensIndex`, we get 0 back for both, because it is the "first owned token" for both Alice and Bob.

Also just like `_allTokensIndex`, the purpose of this data structure is to find a specific tokenID in `_ownedTokens` so we can efficiently remove it (such as when the user transfer or burns token).

_Since these data structures are private, they cannot be directly interacted with. In the next section, we'll understand the functions that read and manipulate these data structure._

## ERC721Enumerable: Functions

According to the ERC721 documentation, the ERC721Enumerable has three public functions:

### `totalSupply()`

![totalSupply() function](https://static.wixstatic.com/media/706568_45fb7b01c7f74e7d96ec53a9bd6aa211~mv2.png/v1/fill/w_740,h_107,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_45fb7b01c7f74e7d96ec53a9bd6aa211~mv2.png)

This function is used to retrieve the total number of NFTs that exist in a contract. It returns the length of the `_allTokens` array.

### `tokenByIndex()`

![tokenByIndex() function](https://static.wixstatic.com/media/706568_2b53497e7f9a442a9693cdf136a736a3~mv2.png/v1/fill/w_740,h_150,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_2b53497e7f9a442a9693cdf136a736a3~mv2.png)

`tokenByIndex` is a simple wrapper around the `_allTokens` array, which takes an index as input and returns the `tokenID` stored at that index in the  `_allTokens` array.

### `tokenOfOwnerByIndex()`

![tokenOfOwnerByIndex() function](https://static.wixstatic.com/media/706568_8127853f3e9f49febfbe8d14c037c896~mv2.png/v1/fill/w_740,h_124,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_8127853f3e9f49febfbe8d14c037c896~mv2.png)

This function is a wrapper around the `_ownedTokens` mapping with some input validation.

![A visual diagram example of an _ownedTokens](https://static.wixstatic.com/media/706568_1d6e7a472132470ca5e4debe29e72601~mv2.png/v1/fill/w_740,h_301,al_c,q_85,enc_auto/706568_1d6e7a472132470ca5e4debe29e72601~mv2.png)

In the above example of the `_ownedTokens` mapping, the address '`0xAli3c3`' owns 3 `tokenIDs`. If the function gets called with this address and an `index` of 2, the tokenID #1 gets returned.

## Adding/Removing tokenIDs From Enumeration

Apart from these functions, OpenZeppelin's ERC721Enumerable implementation features 4 additional private functions, which are used by the `_update` function to ensure the data structures in ERC721Enumerable reflect the current token ownership.

We won’t be going into the details for all of these functions, as they’re not part of the ERC721 specification. However, let’s take a look at one of them:

### `removeTokenFromOwnerEnumeration()`

![_removeTokenFromOwnerEnumeration() function](https://static.wixstatic.com/media/706568_bf949709f7f140f68522220de426059e~mv2.png/v1/fill/w_740,h_305,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_bf949709f7f140f68522220de426059e~mv2.png)

This function is used when a `tokenID` needs to be deleted from an address' enumeration data structures. If an owner sells or burns their NFT, the `tokenID` for that NFT needs to be dissociated from the owner's address, this is where `_removeTokenFromOwnerEnumeration` comes into play.

### The Deletion Process

Before the deletion takes place, the function uses the **_ownedTokensIndex** mapping to check if the `tokenId` is at the last index in the owner’s owned `tokenIDs`. If it is not at the last index, it is swapped with the tokenID at the last index.

This is necessary because if the `tokenID` were to be deleted directly, a gap would be left in the owner’s token-indexes which would cause the `balanceOf()` function to return incorrect results when called with the owner’s address.

After this swap, the function deletes the `tokenID` (which is now the last `tokenID`) from `_ownedTokensIndex` and `_ownedTokens`, effectively removing the token from enumeration.

The rest of such functions in the extension are:

**_addTokenToOwnerEnumeration**: adds a `tokenID` to `_ownedTokens` and `_ownedTokensIndex`, whenever a `tokenID` is minted or transferred to a non-zero address.

It uses the `balanceOf()` function to determine the `index` that can be assigned to the newly minted `tokenID`.

`balanceOf()` will return 3 for an address that owns 3 `tokenIDs`. This means that index #3 can be assigned to a newly minted `tokenID` (since indexing starts from 0).

![_addTokenToOwnerEnumeration() function](https://static.wixstatic.com/media/706568_32278bf97ee844f4b55caa628070ccf2~mv2.png/v1/fill/w_740,h_229,al_c,lg_1,q_85,enc_auto/706568_32278bf97ee844f4b55caa628070ccf2~mv2.png)

**_addTokenToAllTokensEnumeration**: adds a `tokenID` to the data structures tracking all the `NFTs` whenever a `tokenID` is minted, eg., `_allTokensIndex`

![_addTokenToAllTokensEnumeration() function](https://static.wixstatic.com/media/706568_694b331d73b6420391fd119968f5b49b~mv2.png/v1/fill/w_740,h_134,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_694b331d73b6420391fd119968f5b49b~mv2.png)

**_removeTokenFromAllTokensEnumeration**: used when a `tokenID` is burned to keep the data structures updated.

_`_removeTokenFromAllTokensEnumeration`_ _follows a deletion process that is similar to_ _`_removeTokenFromOwnerEnumeration`_.

![_removeTokenFromAllTokensEnumeration() function](https://static.wixstatic.com/media/706568_9c5d8079465d423e98faec61dfe95ceb~mv2.png/v1/fill/w_740,h_283,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_9c5d8079465d423e98faec61dfe95ceb~mv2.png)

## Putting The Pieces Together: The _updateFunction

The _four_ private functions that we briefly learned about in the previous section are used by the `_update` function to mint, burn, or transfer NFTs.

![ERC721 Enumerable _update function](https://static.wixstatic.com/media/706568_7d8bdb639f134960b8a2b8228fd50b1d~mv2.png/v1/fill/w_740,h_462,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/706568_7d8bdb639f134960b8a2b8228fd50b1d~mv2.png)

It is invoked whenever the ownership of a `tokenID` changes. There are two pairs of conditional statements in the function. Let's understand what they're doing:

### <span style="color:#008aff">Conditional Statements</span> #1: <span style="color:#008aff">Checking The Sender Address</span>

The first pair checks if the `tokenID` is being minted or transferred. It handles the removal of a `tokenID` from the previous owner's data structures. Assigning an owner to the `tokenID` is handled in the next conditional statement.

**Case 1: Token is minted**

If it is being minted, it calls `_addTokenToAllTokensEnumeration`, which adds the `tokenID` to `_allTokens` and `_allTokensIndex`.

![A diagram showing the state changing codes t  _addTokensToAllTokenEnumeration](https://static.wixstatic.com/media/706568_2754076fbb8b4ba89940cf5db6bfbd7f~mv2.png/v1/fill/w_740,h_200,al_c,lg_1,q_85,enc_auto/706568_2754076fbb8b4ba89940cf5db6bfbd7f~mv2.png)

**Case 2: Token is transferred**

If it is being transferred, `_removeTokenFromOwnerEnumeration` is called, which removes the `tokenID` from `_ownedTokens` and `_ownedTokensIndex` of the **`previousOwner`** address that the function takes as an input.

![Diagram showing the state changing codes that deletes tokenID from _ownedTokens and _owned TokensIndex in the _removeTokenFromOwnerEnumeration() function](https://static.wixstatic.com/media/706568_730e7706d77a4eef81b912b35f50c531~mv2.png/v1/fill/w_740,h_200,al_c,lg_1,q_85,enc_auto/706568_730e7706d77a4eef81b912b35f50c531~mv2.png)

### <span style="color:Red">Conditional Statements</span> #2<span style="color:Red">: Checking The Receiver Address</span>

The first condition isn't concerned with the address that the `tokenID` is being transferred to. It is the second conditional statement that checks whether the `tokenID` is being burned or transferred to a non-zero address.

**Case 1: Token is burned**

If it is being burned, the `_removeTokenFromAllTokensEnumeration` function is called, which removes the `tokenID` from `_allTokens` and `_allTokensIndex`.

![The Diagram shows the state changing codes that deletes the tokens in _allTokensIndex](https://static.wixstatic.com/media/706568_8394cfdf8ba8435ab9d8450bdcd12892~mv2.png/v1/fill/w_740,h_229,al_c,lg_1,q_85,enc_auto/706568_8394cfdf8ba8435ab9d8450bdcd12892~mv2.png)

**Case 2: Token is transferred**

If it is being transferred to a non-zero address, `_addTokenToOwnerEnumeration` is called, which adds the `tokenID` to `_ownedTokens` and `_ownedTokensIndex` of the **`to`** address.

![Diagram showing the state changing codes that adds tokens to _ownedTokens in _addTokenToOwnerEnumeration() funcion ](https://static.wixstatic.com/media/706568_0fb5156b54c94160a1835d363954ecaa~mv2.png/v1/fill/w_740,h_200,al_c,lg_1,q_85,enc_auto/706568_0fb5156b54c94160a1835d363954ecaa~mv2.png)

## Adding ERC721Enumerable To Your Project

In this section, we'll learn how to add OpenZeppelin's ERC721Enumerable extension to our ERC721 contract in 2 steps.

### 1. Import ERC721Enumerable

At the top of your ERC721 file, add in the following line of code with the rest of your imports:

```solidity
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
```

After that, define the contract in the following manner:

```solidity
contract YourTokenName is ERC721, ERC721Enumerable{

}
```

### 2. Overriding Functions

The inclusion of `ERC721Enumerable` requires some functions from ERC721 to be overridden. These functions are:

1. [_update](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6ba452dea4258afe77726293435f10baf2bed265/contracts/token/ERC721/ERC721.sol#L241-L270)

```solidity
function _update(
    address to,
    uint256 tokenId,
    address auth
) internal override(ERC721, ERC721Enumerable) returns (address) {
    return super._update(to, tokenId, auth);
}
```

2. [_increas](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6ba452dea4258afe77726293435f10baf2bed265/contracts/token/ERC721/ERC721.sol#L224-L228)[e](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6ba452dea4258afe77726293435f10baf2bed265/contracts/token/ERC721/ERC721.sol#L224-L228)[Balance](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6ba452dea4258afe77726293435f10baf2bed265/contracts/token/ERC721/ERC721.sol#L224-L228)

```solidity
function _increaseBalance(address account, uint128 value)
    internal
    override(ERC721, ERC721Enumerable)
{
    super._increaseBalance(account, value);
}
```

3. [supportsI](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6ba452dea4258afe77726293435f10baf2bed265/contracts/token/ERC721/ERC721.sol#L47-L52)[n](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6ba452dea4258afe77726293435f10baf2bed265/contracts/token/ERC721/ERC721.sol#L47-L52)[terface](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/6ba452dea4258afe77726293435f10baf2bed265/contracts/token/ERC721/ERC721.sol#L47-L52)

```solidity
function supportsInterface(bytes4 interfaceId)
    public
    view
    override(ERC721, ERC721Enumerable)
    returns (bool)
{
    return super.supportsInterface(interfaceId);
}
```

**_Note_**_: Other extensions of_ _ERC721_ _that implement a custom_ _`balanceOf()`_ _function (eg._ [_ERC721Consecutive_](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC721/extensions/ERC721Consecutive.sol)_), cannot be used along with the_ _`ERC721Enumerable`_ _extension since they tamper with its functionality._

## Enumeration At A Cost: Caveats Of The ERC721Enumerable Extension

For every transfer, the data structures in `ERC721Enumerable` have to be updated. This makes the contract gas-heavy, adding a considerable amount of gas costs. For projects that must list tokenIDs on-chain however, this is a necessary expense.

## Authorship

This article was written by Poneta, a research intern at RareSkills.

## Learn More With RareSkills

Check out our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp) to learn advanced Solidity concepts.

*Originally Published March 27, 2024*
