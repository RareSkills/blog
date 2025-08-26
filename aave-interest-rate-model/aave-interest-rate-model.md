# The interest rate model of Aave V3 and Compound V2

Interest rates in TradFi (traditional finance) are largely determined by central banks and influenced by market factors. In contrast, DeFi interest rates are algorithmically determined by the demand for loans and the supply of funds from lenders.

Concepts in this article will be explained through the lens of Aave, a leading DeFi lending/borrowing protocol on Ethereum. Compound V2's interest rate model is identical, with minor implementation differences. You may encounter differences when comparing the implementations of other lending/borrowing protocols.

A lending protocol uses smart contracts to aggregate liquidity from lenders, and allow for loans to be taken against this combined liquidity by borrowers. We sometimes refer to the liquidity as “funds” or “capital.”

Lenders (a.k.a Suppliers) will earn interest based on the liquidity they provide. Borrowers will be obligated to pay interest on their loan.

Lending protocols often set aside a “reserve factor” — a percentage of capital supplied by lenders that they do not receive interest on. The interest from this capital instead goes to the protocol, and is sometimes called the “spread.”

The degree of borrowing influences interest rates — we call this “borrow demand.” The greater the percentage of deposits loaned out, the more the interest rates rise. We refer to the degree of borrowing relative to the supply of borrowable funds as “utilization.” If there are no borrowers, utilization is 0% and if the deposits are fully loaned out, utilization is 100%. The value of utilization is used as the sole driver of interest rates.

The following diagram illustrates this — we will explain each step throughout this article.

![borrow demand determines utilization determines borrow interest rates determines supply interest rates](https://static.wixstatic.com/media/935a00_4479dfe0d8774b95bdf66a3c2a4c5a1c~mv2.png/v1/fill/w_418,h_78,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_4479dfe0d8774b95bdf66a3c2a4c5a1c~mv2.png)

### Authorship

This article was written by [Calnix](https://twitter.com/cal_nix) with animations created by [Aymeric Taylor](https://twitter.com/TaylorAymeric).

## Utilization Formula

As mentioned earlier, Utilization is determined by borrow demand.

![borrow demand determines utilization](https://static.wixstatic.com/media/935a00_eb2c9185a1ff442099c5d7c571dd793f~mv2.png/v1/fill/w_350,h_53,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_eb2c9185a1ff442099c5d7c571dd793f~mv2.png)

Utilization is typically calculated as such:

![utilization = borrowed capital divided by total capital](https://static.wixstatic.com/media/935a00_ca27d1f3a4454ccc92e5701a1553c50c~mv2.png/v1/fill/w_455,h_48,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_ca27d1f3a4454ccc92e5701a1553c50c~mv2.png)

Note that borrow demand is relative to total deposits. If the total deposits increases but borrow demand stays the same, utilization goes down.

## How utilization drives borrow interest rates

Utilization in turn determines borrow interest rates

![utilization determines borrow interest rates](https://static.wixstatic.com/media/935a00_8403dcd10dd0438b92c07a53cda09008~mv2.png/v1/fill/w_350,h_51,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_8403dcd10dd0438b92c07a53cda09008~mv2.png)

In a properly designed protocol, changes in utilization lead to rising/falling interest rates as explained below.

When utilization rises: **(indicative of high demand for borrowing)**

-   Borrow and supply interest rates increase
-   Lending is incentivized and borrowing is disincentivized
-   As a consequence, idle liquidity increases, and borrow demand falls, leading to a lower level of utilization.

When utilization falls: **(indicative of excess idle liquidity)**

-   Borrow and supply rates decrease
-   Borrowing is incentivized and lending is disincentivized
-   As a consequence, idle liquidity decreases, and borrow demand rises; leading to a higher level of utilization.

![linear interest rate curve](https://static.wixstatic.com/media/935a00_d161d6059bc24bf4b707a1e3a68f1f02~mv2.png/v1/fill/w_740,h_429,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_d161d6059bc24bf4b707a1e3a68f1f02~mv2.png)

The function that translates the utilization level to the borrow interest rate is defined by the **interest rate model**. The parameters are usually set by protocol governance.

## Calculating the relationship between the borrow interest rate and the supply interest rate

We now look at the final relationship in the chain

![borrow interest rates determine supply interest rates](https://static.wixstatic.com/media/935a00_00bc13d3272d4b8ab52a48bf65065fc5~mv2.png/v1/fill/w_437,h_53,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_00bc13d3272d4b8ab52a48bf65065fc5~mv2.png)

The relationship is determined as follows:

![supply interest rate = borrow rate x utilization](https://static.wixstatic.com/media/935a00_0dc93a74e5584a3da7e445b3eaac2910~mv2.png/v1/fill/w_488,h_52,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_0dc93a74e5584a3da7e445b3eaac2910~mv2.png)

Not all lending protocols set the supply interest rate using this formula. Notably, Compound V3 calculates the supply interest rate directly as a function of utilization and does not factor in the borrow rate.

**Example:** Consider a lending protocol where the borrow rate is 10% and the utilization level is 50%. In this case, the interest rate enjoyed by depositors would be:

-   Supply Rate = Borrow Rate × Utilization level
-   Supply Rate = 10% × 50% = 5%

Depositors would earn an interest rate of 5% on their deposits.

Below we see a plot of <span style="color:#008aff">supply interest rate</span> as a function of the <span style="color:Green">borrow interest rate</span>. When utilization is zero, lenders don't earn any interest.

![supply interest rate curve in blue](https://static.wixstatic.com/media/935a00_75b7326518b544b3bbe353d0d605b19f~mv2.png/v1/fill/w_740,h_429,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_75b7326518b544b3bbe353d0d605b19f~mv2.png)

## What is the motivation for multiplying borrow interest with utilization?

![supply interest rate = borrow interest rate x utilization](https://static.wixstatic.com/media/935a00_924518fd0c304b9498e05c4de712548e~mv2.png/v1/fill/w_517,h_55,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_924518fd0c304b9498e05c4de712548e~mv2.png)

If 50% of the funds are borrowed, only half of the capital provided by lenders is lent. If the borrower pays 10% on half the assets available, the lender gets 5% on their capital. If we paid the lenders 10% on their entire capital, but only half of it was borrowed, then the protocol would not be able to meet its obligations.

Consider the case where $100 is supplied to the protocol and $50 is borrowed. If the borrower pays 10% interest, they would pay $5 to the protocol. But if the protocol pays the suppliers 10% interest on the $100, it would owe them $10 — which it hasn’t earned from borrowers.

## Adding in a reserve factor: accounting for spread

While the previous example is useful for our understanding, it ignores the spread we mentioned at the start - the cut that goes to the protocol, which could reflect fees, treasury contribution, etc.

![supply interest rate = borrow interest rate x utilization x (1 - reserve factor)](https://static.wixstatic.com/media/935a00_7cbf5ccbd4034135831ff6c79b3364d6~mv2.png/v1/fill/w_612,h_50,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_7cbf5ccbd4034135831ff6c79b3364d6~mv2.png)

The reserve factor allocates a portion of the borrow interest to the protocol, with the rest divvied out to depositors.

**Worked Example:** Assume a lending protocol where the borrow rate is 10%, utilization level is 50% and reserve factor 20%. In this case, the interest rate enjoyed by depositors would be:

-   Supply Rate = <span style="color:Red">Borrow Rate</span> × <span style="color:#008aff">Utilization level</span> × (1 - <span style="color:#8d28a4">Reserve Factor</span>)
-   Supply Rate = <span style="color:Red">10%</span> × <span style="color:#008aff">50%</span> × (1 - <span style="color:#8d28a4">20%</span>) = 5% × (80%) = 4%

As you can see, a fifth of the borrow interest is siphoned off, and depositors now earn 4% on their deposits.

## Real interest rate models

DeFi applications try to encourage the utilization to be around a certain percentage, usually between 80-95%, this is called the “optimal utilization.”

It is undesirable for all the available funds to be lent out (100% utilization). Under this circumstance, lenders cannot withdraw. Therefore, when all the capital is used up, we want strong incentives in place for lenders to supply more funds to the protocol — or for borrowers to return borrowed assets. Therefore, the supply curve is typically piecewise linear and “kinked” — it rises slowly up to the optimal utilization point, then rises faster.

### Formula for piecewise linear kinked function

Any interest rate model must be a function of Utilization **𝑈** since

![utilization determines borrow interest rates](https://static.wixstatic.com/media/935a00_acae9a52be1042acbbe5fcef424eaef3~mv2.png/v1/fill/w_350,h_51,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_acae9a52be1042acbbe5fcef424eaef3~mv2.png)

Aave’s piecewise linear kinked interest rate model is as follows.

![Aave interest rate formula](https://static.wixstatic.com/media/935a00_6c64ca8ac8d84d98b4300e3bccb638b9~mv2.png/v1/fill/w_675,h_91,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_6c64ca8ac8d84d98b4300e3bccb638b9~mv2.png)

To get an intuition for the formula, we can rearrange the equations in the form of y = mx + c

![Aave interest rate formula y = mx + c animation](https://static.wixstatic.com/media/935a00_45b41860f6fd4d79aef63eabf238f563~mv2.gif)

The following animation illustrates what each part of the formula does

![Aave interest rate formula animated](https://static.wixstatic.com/media/935a00_c08cc9b5094f457aa4999978cc81e1a2~mv2.gif)

Note that **𝑈** is the independent variable here, `Rborrow` is a function of that and other parameters that are determined by protocol governance.

The "kink" area of the curve is what the protocol considers "optimal." If the utilization exceeds that level, the interest rates start rising quickly to incentivize lenders to supply more capital or borrowers to pay down loans.

### DAI example

Let’s look at Aave’s DAI interest rate model, as defined by the contract `DefaultReserveInterestRateStrategy` on Ethereum. The four essential parameters (`U_optimal`, `R_intercept`, `R_slope1`, `R_slope2`) are publicly accessible from the smart contract.

Here is the smart contract address: [https://etherscan.io/address/0x694d4cFdaeE639239df949b6E24Ff8576A00d1f2](https://etherscan.io/address/0x694d4cFdaeE639239df949b6E24Ff8576A00d1f2)

![dai etherscan Aave interest rate model screenshot](https://static.wixstatic.com/media/935a00_d8c82fd1ee3446c69f0aa74c15899d3f~mv2.png/v1/fill/w_740,h_1305,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_d8c82fd1ee3446c69f0aa74c15899d3f~mv2.png)

From the on-chain values, we can see that the following parameters were used in the interest-rate model:

-   U_optimal ( <span style="color:#008aff">blue arrow</span> ) has the variable name OPTIMAL_USAGE_RATIO = 0.80
-   R_intercept ( <span style="color:Red">red arrow</span> ) has the public function name getBaseVariableBorrowRate = 0
-   R_slope1 ( <span style="color:Green">green arrow</span> ) has the public function name getVariableRateSlope1 = 0.04
-   R_slope2 ( <span style="color:#8d28a4">purple arrow</span> ) has the public function name getVariableRateSlope2 = 0.75

The values are represented as [fixed point](https://www.rareskills.io/post/solidity-fixed-point) Ray numbers (27 decimals, i.e. 1 = 1e27).

_Parameters are accurate at time of writing._

## Conclusion

Interest rates are a function of the utilization of the assets in the protocol. The exact shape of the function is set by governance. The interest suppliers earn is less than what the borrowers pay due to the spread and the reserve factor.

## Learn More with RareSkills

Learn more technical topics in DeFi and web3 in our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps).

*Originally Published December 1, 2023*
