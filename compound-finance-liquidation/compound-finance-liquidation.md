# Understanding Collateral, Liquidations, and Reserves in Compound V3

In this chapter we will examine the following topics about Compound V3:

-   collateral valuation
-   absorbing insufficiently collateralized loans (liquidations)
-   selling absorbed collateral
-   what reserves are
-   and how reserves affect liquidations.

These are a lot of topics to go over in one article, but they are all heavily intertwined, so it’s best we cover them in one treatment.

## Prerequisites

The reader should already be familiar with how [Compound V3 defines principal and present value](https://www.rareskills.io/post/defi-interest-rate-indexes) and with [DeFi liquidations and collateral](https://www.rareskills.io/post/defi-liquidations-collateral).

## UserBasic Storage Struct

Let’s revisit the [UserBasic struct in CometStorage.sol](https://github.com/compound-finance/comet/blob/main/contracts/CometStorage.sol#L29-L35).

![UserBasic struct](https://static.wixstatic.com/media/935a00_7c29ee89b4c348dc91cde68a0c6aab32~mv2.png/v1/fill/w_315,h_191,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_7c29ee89b4c348dc91cde68a0c6aab32~mv2.png)

If the `principal` (<span style="color:#008aff">blue box</span>) is negative, it means the user is a borrower, and the negative value will be the **_principal_** value of their debt.

`assetsIn` (<span style="color:Red">red box</span>) is a bitmap to indicate whether they’ve deposited a certain collateral asset or not. At this time of writing, the bitmap is laid out as follows:

![compound collateral asset bitmap](https://static.wixstatic.com/media/935a00_7995fe43f6c0485e9dc5c9d68aad8327~mv2.png/v1/fill/w_315,h_231,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_7995fe43f6c0485e9dc5c9d68aad8327~mv2.png)

The variables `baseTrackingIndex` and `baseTrackingAccrued` are for bookkeeping the distribution of rewards, and will be discussed in a separate article. The variable `_reserved` is unused.

Note that this struct does not tell us how much collateral the user is holding. That is held in `balance` variable in the [UserCollateral struct](https://github.com/compound-finance/comet/blob/main/contracts/CometStorage.sol#L37-L40) which is stored in the `userCollateral` nested mapping. The `_reserved` variable is unused.

![UserCollateral struct](https://static.wixstatic.com/media/935a00_8d35628663774b9fb71c33534eae994a~mv2.png/v1/fill/w_666,h_220,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_8d35628663774b9fb71c33534eae994a~mv2.png)

To list out the collateral a user has supplied, we loop through `0…numAssets` Compound stores and check if that bit is set to one for that user. If it is, we obtain the token address associated with that bit and check the user balance in `userCollateral[user][collateralAsset]` to see how much of that collateral the user holds.

By multiplying the `balance` with the oracle price, we know the dollar value of the user’s collateral. The following table gives an example of summing up the total value of a user’s collateral.

![example total collateral value](https://static.wixstatic.com/media/935a00_4baade0264c34566aeeeebd619c41392~mv2.png/v1/fill/w_666,h_167,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_4baade0264c34566aeeeebd619c41392~mv2.png)

## AssetInfo

The address of the oracle where Compound obtains the collateral price from is stored in the AssetInfo struct (<span style="color:#008aff">blue box</span>).

![AssetInfo struct](https://static.wixstatic.com/media/935a00_6d525717eee34f6ea4deaef6078d6719~mv2.png/v1/fill/w_315,h_201,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_6d525717eee34f6ea4deaef6078d6719~mv2.png)

Observe the `AssetInfo` struct above is 432 bits large — it takes 2 slots to store it. We will revisit this in a following section.

## Displaying AssetInfo on the Compound Finance Markets

Let’s compare the content of the `AssetInfo` struct shown above with the Compound Finance Market UI. Here we see most of the information displayed.

The documents and code do not tell us what the variable `scale` is used for, but it holds the number 1e18, so presumably it is to let the consumer know how to scale the percentages.

The meaning of these variables was explained in the article on [liquidations and collateral](https://www.rareskills.io/post/defi-liquidations-collateral).

![AssetInfo side by side with the Compound V3 markets UI](https://static.wixstatic.com/media/935a00_8695faaefe664a8387eae61085051a33~mv2.jpg/v1/fill/w_666,h_195,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_8695faaefe664a8387eae61085051a33~mv2.jpg)

When we query the current values for the token UNI (assetId 3) ([https://etherscan.io/address/0xc3d688B66703497DAA19211EEdff47f25384cdc3#readProxyContract#F16](https://etherscan.io/address/0xc3d688B66703497DAA19211EEdff47f25384cdc3#readProxyContract)), we can compare the values to the ones displayed on the marketplace. The relationship should be clear. Of note: the liquidation penalty is `1 - liquidation factor`. The `liquidateCollateralFactor` is the LTV at which the loan gets liquidated. The `liquidationFactor` encodes the liquidation penalty. The fact that the `liquidationFactor` in the struct does not mean the same thing as the `liquidationFactor` in the UI is confusing.

The top image is the screenshot, and the values below is a screenshot from Etherscan querying `getAssetInfo()` for token UNI.

Below we show the relationship between the UNI collateral parameters on the Compound UI and Etherscan querying `getAssetInfo()` for token UNI.

![Compound V3 Market data screenshot on top of the function return from getAssetInfo()](https://static.wixstatic.com/media/935a00_85288f90ec4f436da0ba0280ab52ff21~mv2.jpg/v1/fill/w_666,h_432,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_85288f90ec4f436da0ba0280ab52ff21~mv2.jpg)

The obvious next question is, “where does Compound store the `AssetInfo` structs?”

## AssetInfo “structs” are kept in immutable variables

Each asset’s info is packed into immutable variables — it is not kept in storage for gas efficiency purposes. Since it takes two 32 byte words to store the `AssetInfo` struct, Comet numbers off the uint256 words with `assetXX_a, assetXX_b`. The `XX` here indicates the asset index. So `asset00_a` and `asset00_b` collectively hold the `AssetInfo` struct for asset 0. Remember, it takes two 256 bit variables to store AssetInfo, which is 432 bits large.

![AssetInfo immutable variable layout](https://static.wixstatic.com/media/935a00_2c0947b8f34648c988799fa9b01e2b5a~mv2.png/v1/fill/w_666,h_300,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_2c0947b8f34648c988799fa9b01e2b5a~mv2.png)

We can now show the implementation of `getAssetInfo()` from [Comet.sol:280-356](https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol#L280-L356). It simply unpacks the immutable variable into the `AccountInfo` struct and returns it. The bitshifting and packing it uses is straightforward, so we will not explain it here.

![getAssetInfo function](https://static.wixstatic.com/media/935a00_12c8bb252ece4700a73e53f189160e76~mv2.png/v1/fill/w_666,h_1406,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_12c8bb252ece4700a73e53f189160e76~mv2.png)

Because these variables are immutable, [governance](https://www.rareskills.io/post/governance-contract-solidity) must deploy a new implementation and update the proxy if it wishes to add another collateral asset or change the parameters of one of the assets. Care must be taken to only append assets and not interfere with previous definitions.

## Checking if a borrower can be liquidated

The Comet.sol [isLiquidatable() function](https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol#L541-L581) sums up the collateral assets held by a user multiplied by their liquidationFactor. If this sum is less than the present value of their debt (which is a negative number), then the user is liquidatable.

![isLiquidatable formula](https://static.wixstatic.com/media/935a00_b3a8256e666c453cbb62e180ec8cd0d4~mv2.png/v1/fill/w_666,h_60,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_b3a8256e666c453cbb62e180ec8cd0d4~mv2.png)

This means a borrower might have one asset below the liquidation threshold, but if other collateral assets balance it out the deficit, then the user is not liquidatable.

The full value of the collateral does not “count” towards the user’s collateral balance — it is reduced by the liquidation factor.

Here is the same example from earlier showing the true value of the user’s collateral assets.

![User collateral value scaled by liquidation threshold](https://static.wixstatic.com/media/935a00_279fe8efdaac4c32bafbc96342cd0771~mv2.png/v1/fill/w_666,h_173,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_279fe8efdaac4c32bafbc96342cd0771~mv2.png)

In the example above, the hypothetical borrower will get liquidated if their loan balance exceeds $8,360.

## Liquidating a borrower (absorb)

If the `isLiquidatable()` function returns true, then the borrower’s collateral can be absorbed into the protocol. What some protocols call “liquidation” Compound V3 calls “absorb.”

Absorbs are all or nothing — there is no option to liquidate part of the collateral in Compound V3. The entirety of the borrower’s asset balances are set to zero.

### Example absorb

Suppose Bob deposited \$1000 of ETH into Compound V3 and borrowed \$800 USDC. This satisfies the collateralization ratio of 80%. The value of ETH drops to \$880 causing the LTV (loan to value) to hit 90.9%, triggering the 90% liquidation threshold.

A liquidator calls `absorb()` on Bob’s account and the \$880 of ETH collateral are absorbed into the protocol.

Let’s say the liquidation penalty is 5%.

Since the collateral value is currently \$880 ETH, 5% of that is \$44 in ETH.

The protocol will deduct $44 from Bob’s collateral as a penalty, leaving \$836. Since Bob borrowed \$800 USDC, there is a \$36 surplus. That is, \$800 is taken by the protocol to cover the debt leaving \$36 left over. This is credited to Bob who now becomes a lender with a \$36 USDC deposit.

Bob has already withdrawn the \$800 USDC when he took out the loan, so his total holdings are now $836.

Note that nothing in the `absorb()` interaction directly rewarded the liquidator.

Note that **when a borrower gets liquidated, they will become a lender if the collateral is sufficient to cover the debt.**

If the collateral is not sufficient to cover the debt, then the protocol implicitly takes a loss from its _reserves_ which we discuss next.

## Reserves

Interest paid by borrowers that is in excess of what lenders earned are called “reserves” in Compound V3.

### Reserves Example

Alice lends the protocol 100 USDC and earns 5% interest. Bob borrows 100 USDC from the protocol and pays 10% interest. For the sake of simplicity, let’s assume Alice and Bob are the only actors in the system. There will be an extra 5% interest that Bob paid that Alice didn’t earn. This extra amount is the reserve.

It doesn’t matter whether Bob has paid back the loan or not (i.e. has he transferred 110 USDC to the protocol yet or not). He owes the protocol 110 USDC and the protocol owes Alice 105 USDC. Therefore, there are 5 USDC in reserves.

Suppose Bob pays off the loan. Now the protocol has a balance of 110 USDC, 105 of which is due to Alice. There are still 5 USDC in reserves — nothing changed.

The function `getReserves()` returns this value. The USDC the protocol “owns” is the sum of

1) the balance of USDC held by Compound i.e. `ERC20(baseToken).balanceOf(address(this))` and

2) the present value of the `totalBorrow`,

3) minus the amount the protocol owes to lenders, i.e. the present value of the `totalSupply`.

In other words, it is `usdc_balance + totalBorrow - totalSupply`.

As you can see in the function below, `totalSupply` is assigned a negative sign because that is what Compound owes to the lenders. The positive factors — the amount of held USDC and the net amount of USDC owed to Compound from borrowers — are the amount of USDC “owned” by the Compound.

![getReserves() function](https://static.wixstatic.com/media/935a00_2efd83b300b24ef08b0d5582bad438c1~mv2.png/v1/fill/w_666,h_187,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_2efd83b300b24ef08b0d5582bad438c1~mv2.png)

If we look at the [getReserves() function on Etherscan](https://etherscan.io/address/0xc3d688B66703497DAA19211EEdff47f25384cdc3#readProxyContract), we will see the reserves at the time of writing are 3.47 million USD (6 decimals).

![getReserves showing 3.47 million dollars on Etherscan](https://static.wixstatic.com/media/935a00_fc421dd1eb8d40b39be25916093a0b57~mv2.png/v1/fill/w_315,h_108,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_fc421dd1eb8d40b39be25916093a0b57~mv2.png)

When we look at the [USDC / Mainnet Compound market](https://app.compound.finance/markets/usdc-mainnet), we also see the front-end displaying the current reserves as 3.47 million.

![Compound V3 Markets UI showing 3.47 million dollars in reserve](https://static.wixstatic.com/media/935a00_ae2f77c7ddde4622b2be1129cc43b752~mv2.png/v1/fill/w_666,h_405,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_ae2f77c7ddde4622b2be1129cc43b752~mv2.png)

If we call `getReserves()` before and after an `absorb()` we would notice that the reserves decreased. This happens for two reasons:

1) The protocol paid off the loan (so less is owed to it). This came out of the reserves, so naturally there are less reserves.

2) The borrower becomes a lender with a small deposit. This deposit is owed by Compound to the lender, further decreasing the reserves.

## withdrawReserves()

The excess reserve is for use by governance and can be withdrawn using the function below.

![withdrawReserves() function](https://static.wixstatic.com/media/935a00_0e5d11011eb3446e8650401daf84af81~mv2.png/v1/fill/w_666,h_336,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_0e5d11011eb3446e8650401daf84af81~mv2.png)

## Target Reserves

Compound has a public immutable variable [targetReserves defined in Comet.sol](https://github.com/compound-finance/comet/blob/22cf923b6263177555272dde8b0791703895517d/contracts/Comet.sol#L91-L92)

![targetReserves immutable variable](https://static.wixstatic.com/media/935a00_4229141b7255415ea0c07b5221f4b617~mv2.png/v1/fill/w_666,h_103,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_4229141b7255415ea0c07b5221f4b617~mv2.png)

When we look at the [targetReserves on Etherscan](https://etherscan.io/address/0xc3d688B66703497DAA19211EEdff47f25384cdc3#readProxyContract), we see it is 5 million USDC.

![An image of targetReserves on Etherscan](https://static.wixstatic.com/media/935a00_520ffababb864d359d0ac192a78fd118~mv2.png/v1/fill/w_315,h_80,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_520ffababb864d359d0ac192a78fd118~mv2.png)

Target reserves has only one use in the protocol: to determine if the protocol has enough “margin of safety” to not sell absorbed collateral. Note that governance could change this value by deploying a new Comet instance.

That is, **if Compound V3 has sufficient “excess cash,” they’d rather hold on to the collateral for the purpose of speculating that it will increase in value.**

Let’s examine the only function where this variable is used.

## buyCollateral()

Collateral is still inside the protocol after an absorb. All that happened was that the user’s collateral balance was set to zero — however the collateral was not transferred anywhere. This collateral is still “inside” Compound V3.

To incentivize liquidators, collateral held by Compound is sold at a discount via the [buyCollateral() function](https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol#L1195).

There are two crucial pieces of business logic in this function:

1.  If the reserves amount is larger than the target reserves ($5 million) this function will revert, not allowing liquidators to purchase collateral. (<span style="color:#c1c146">yellow box</span> in the code below). As mentioned above, Compound wishes to speculate on the collateral. Since it is already in a cash-heavy position, they don’t wish to accumulate more cash.

2.  The exchange rate the protocol sells the collateral at is determined by the `quoteCollateral()` function (<span style="color:Red">red box</span> in the code below).


The rest of the code should be self-explanatory.

![buyCollateral function](https://static.wixstatic.com/media/935a00_d70b9c7935e94bd5830da8f10f687bce~mv2.png/v1/fill/w_666,h_399,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_d70b9c7935e94bd5830da8f10f687bce~mv2.png)

## Liquidation bot

To liquidate a borrower, the liquidator calls `absorb()` with the borrower’s account as arguments, then calls `buyCollateral()` in the same transaction. The liquidator should check that reserves are not more than the target reserves, and that the account is liquidate able via `isLiquidateable()`. Compound V3 provides a [reference liquidator bot](https://github.com/compound-finance/comet/tree/main/contracts/liquidator). Keep in mind this is a reference implementation — trying to get liquidations before others is highly competitive, so your code will need to be extremely [gas optimized](https://www.rareskills.io/post/gas-optimization) to be able to profitably liquidate a position before others do.

## Summary

The list of assets Compound V3 accepts as collateral — as well as their parameters such as collateralization ratio, liquidation ratio, oracle address, and so on, are “packed” in immutable variables. To change these, a proxy upgrade to Compound V3 is necessary.

A user’s collateral balance is tracked via a combination of a bitmap indicating if they have a non-zero balance for a certain asset, then a nested mapping of borrower ⇒ asset ⇒ assetBalance. Their total collateral balance is each the sum of each asset multiplied the oracle price.

Liquidations are all-or-nothing. When a user is liquidated, they will lose `1 - liquidationRatio` of their collateral and the leftover will be used to pay off the debt and assign the user a positive balance.

The protocol now holds excess collateral and makes this available for sale at a discount according to the `quoteCollateral()` function. It will not sell however if the reserves are higher than the `targetReserves`.

Reserves are simply the money owed to the protocol plus the USDC balance of the protocol minus the amount the protocol owes to lenders. This money is withdrawable by governance.

## Learn More with RareSkills

See our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) for advanced technical web3 courses.

*Originally Published January 8, 2024*
