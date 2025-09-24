# cUSDC V3 (Compound V3) as a non-standard Rebasing Token, CometExt.sol

The Compound V3 contract behaves like a rebasing ERC-20 token. A rebasing token is a token which has an algorithmically adjusted supply rather than a fixed one. The “token” here represents the present value of positive USDC balances. That is, lenders can transfer the present value of their principal to other addresses as if it were an ERC-20 token. Since the principal value is generally increasing due to interest accrual, this ERC-20 token is rebasing upwards over time.

**Compound V3 does not use a token vault standard (e.g. [ERC-4626](https://www.rareskills.io/post/erc4626)) to track “shares” of the lending pool.**

As we noted in our discussion of principal and present value, a user may have deposited 100 USDC but have a credit of 110 USDC due to interest accrued — the 110 is the present value. It is this unit of account that the ERC-20 functionality of Compound V3 manages.

## Prerequisites

Users must be familiar with interest indexes and Compound V3’s notion of [present value and principal value](https://www.rareskills.io/post/defi-interest-rate-indexes). The terms Compound V3 and Comet are used interchangeably in this article since Comet is the name of the primary smart contract which has the functions we discuss here.

## Structure of this article

Each heading will discuss an ERC-20 function that Compound V3 implements, and how it implements the function. Some of the functions are not part of the ERC-20 standard but are relevant to this discussion.

## totalSupply and totalBorrow (Comet.sol)

The `totalBorrow`, as the name implies, is the total amount of USDC borrowed. That is, it is the present value of the debt. We can corroborate that with what the contract returns and what we see on the Compound platform. This is not just the amount of USDC borrowers withdrew from the platform — it includes the interest accrued on the borrowed USDC.

Below we show a screenshot of the Compound V3 UI showing this value and Etherscan returning the result of the `totalBorrow()` function.

![total borrowing screenshot compound v3](https://static.wixstatic.com/media/935a00_36c539af3e6d492bb8ada32131daebdd~mv2.png/v1/fill/w_666,h_170,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_36c539af3e6d492bb8ada32131daebdd~mv2.png)

Similarly, the totalSupply() is not the amount of USDC lenders deposited into Compound — it is the _present value_ of the total deposits.

The [totalSupply and totalBorrow code](https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol#L1267C1-L1285C6) screenshotted below should make the relationship to the present value clear.

![totalSupply() and totalBorrow() functions](https://static.wixstatic.com/media/935a00_1daf2c3dffb24611aaf6de16b6554213~mv2.png/v1/fill/w_666,h_352,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_1daf2c3dffb24611aaf6de16b6554213~mv2.png)

Below we have screenshotted two queries of totalSupply. Note that the `totalSupply` has increased in the second screenshot on the right.

![totalSupply() increasing](https://static.wixstatic.com/media/935a00_c208f71cb0d942129f825de5a37d9689~mv2.png/v1/fill/w_666,h_251,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_c208f71cb0d942129f825de5a37d9689~mv2.png)

**The totalSupply() function behaves as totalSupply() from ERC-20.**

Borrowers are not able to transfer debt, so totalBorrow is not used for any token-like interfaces.

## balanceOf (Comet.sol)

BalanceOf was already covered in our discussion about principal and present value, so we won’t repeat it here.

## transfer and transferFrom (Comet.sol)

Both transfer and transferFrom will transfer the present value of the lender. The amount parameter is measured in present value.

![transfer() and transferFrom()](https://static.wixstatic.com/media/935a00_084e592a61c24a2f96c90b17f2b2d2c3~mv2.png/v1/fill/w_666,h_394,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_084e592a61c24a2f96c90b17f2b2d2c3~mv2.png)

Both these functions call `transferInternal` under the hood. The right way to transfer the entire balance in Compound V3 is to transfer uint256 max value. Specifying the entire balance can be a bit tricky because it increases every second.

![transferInternal()](https://static.wixstatic.com/media/935a00_fe5751fa0f65435ebb84452e1e0cf3c3~mv2.png/v1/fill/w_666,h_291,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_fe5751fa0f65435ebb84452e1e0cf3c3~mv2.png)

Note that there is no mechanism for borrowers to transfer collateral to other addresses, as `transferCollateral` is only internal. This function is used for liquidations.

## The remainder of the ERC-20 functions are in CometExt.sol

Because of the 24 KB deployment limit, Comet splits off some of its functionality to CometExt.sol using the [fallback extension pattern](https://www.rareskills.io/post/fallback-extension-pattern). The majority of the functions in CometExt are related to the ERC-20 functionality.

## approve (CometExt.sol)

The approve() functionality for cUSDCv3 is non-standard in that it only accepts [type(uint256).max](https://www.rareskills.io/post/uint-max-value-solidity) or zero. Since the balance of an account is constantly changing due to the rebasing, it isn’t possible to give someone an allowance for exactly the entire balance, since the balance keeps increasing as interest accumulates.

![approve()](https://static.wixstatic.com/media/935a00_b5a2f63e5d84476eb21b2c428bebbff6~mv2.png/v1/fill/w_666,h_356,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b5a2f63e5d84476eb21b2c428bebbff6~mv2.png)

`approve()` calls allowInternal under the hood which is worth examining separately

### allow() and allowInternal()

The function allow() not part of ERC-20, but it behaves the same as approve() in that it gives an address maximum allowance. Both approve() and allow() use allowInternal() under the hood to accomplish giving an address maximum allowance.

Because approve is essentially binary, allow behaves like approve except it takes a boolean argument to give approval for the full value instead of a uint256.

![allow() and allowInternal() functions](https://static.wixstatic.com/media/935a00_63a4f631ea0c4d7997e0cca4a736f711~mv2.png/v1/fill/w_666,h_318,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_63a4f631ea0c4d7997e0cca4a736f711~mv2.png)

The “allowances” storage variable is kept in [CometStorage.sol](https://github.com/compound-finance/comet/blob/main/contracts/CometStorage.sol) as `isAllowed`.

![isAllowed() storage variable](https://static.wixstatic.com/media/935a00_0f8d1fce0e3e443390300a964d392f17~mv2.png/v1/fill/w_666,h_103,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_0f8d1fce0e3e443390300a964d392f17~mv2.png)

Allowance is all or nothing in Compound V3. That’s why `approve` only accepts the maximum uint256 value. There is no storage variable for storing allowance as a number like traditional ERC-20 tokens do.

## allowance (CometExt.sol)

Allowance is binary, you only have approval for the max uint256 value or zero. Any other value will revert.

`hasPermission` simply returns a Boolean value signifying an address has unlimited approvals or none at all.

## allowBySig() is a non-standard ERC-20 permit() function

CometStorage exposes `mapping(address => uint256)` public userNonce as a public variable rather than the `nonces(address owner) external returns (uint)` specified by [EIP 2612](https://eips.ethereum.org/EIPS/eip-2612).

## name() and symbol() (CometExt.sol)

The functions `name()` and `symbol()` are optional ERC-20 functions that return strings.

[CometExt.sol](https://github.com/compound-finance/comet/blob/main/contracts/CometExt.sol) does not store these values in string variables, but instead in immutable `bytes32` variables for gas efficiency purposes. Solidity does not allow for casting `bytes32` to strings directly, so the immutable variables are converted to strings on the fly using the code below.

One lesser known detail about `bytes1`, `bytes2`, …, and `bytes32` datatypes in Solidity is that they can be indexed at the byte level like byte arrays. For example,

```solidity!
contract Example {
    bytes32 immutable x = 0x3300000000000000000000000000000000000000000000000000000000000000;

	function main() external pure returns (bytes1) {
        return x[0]; // returns 0x33
    }
}
```

CometExt initializes a bytes array in memory and copies over the characters stored in the `name32` and `symbol32` bytes32 immutable variables, then casts that to a string.

![name() and symbol() functions](https://static.wixstatic.com/media/935a00_f57d93d9dd264483b8719d431dfdcd32~mv2.png/v1/fill/w_666,h_934,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_f57d93d9dd264483b8719d431dfdcd32~mv2.png)

## Calling functions in CometExt

The ABI known to Etherscan does not know of these functions, so they will not appear on the list of Comet functions. However, they can be called since they will hit the Comet fallback and be delegated to the CometExt contract.

Here is an example calling `name()` and `symbol()` using Foundry’s cast.

![cast call foundry](https://static.wixstatic.com/media/935a00_8c8667f143114295a485cd85cb339f24~mv2.png/v1/fill/w_666,h_43,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_8c8667f143114295a485cd85cb339f24~mv2.png)

## Conclusion

CometV3 behaves like a rebasing ERC-20 token that represents the positive balances of lenders. This positive balance can be transferred to other addresses like a regular ERC-20 token. The `approve()` function is non-standard — it can only do an infinite approve or none at all. Similarly, the `permit()` function for gasless approvals is non-standard.

## Learn More with RareSkills

See our blockchain bootcamp for more.

*Originally Published January 7, 2024*
