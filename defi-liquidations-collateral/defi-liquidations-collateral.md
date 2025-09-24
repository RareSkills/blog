# DeFi Lending: Liquidations and Collateral

In TradFi, when someone defaults on a loan, the creditor has the right to seize assets or garnish wages.

In DeFi, when someone defaults on a loan it isn’t possible to take assets from their wallet. To enforce repayment, users must deposit a larger notional value of cryptocurrency than they are borrowing.

For example, a user must deposit $1,000 of Ether before they can borrow $800 of USDC. The Ether in this case is called the “collateral.” Depositing more collateral than the borrowed amount is called “overcollateralization”.

## **Motivation for DeFi Borrowing**

A common use for DeFi borrowing is leverage. One can deposit $1000 of ETH, borrow $800 of USDC, then use the USDC to buy $800 ETH. Now they effectively own $1,800 ETH and will have significantly more upside if ETH rises in price (along with more risk on the downside).

Converting ETH to dollars is considered a taxable event in some jurisdictions, so borrowing USDC using ETH as collateral (then paying the loan off later) may be cheaper than swapping the assets.

More generally, taking a loan against an asset (a car, house, etc) for short term liquidity needs is fairly common, so DeFi lending protocols help fulfill this use case.

## Collateral factor

The collateral factor is the percentage of the collateral value that can be borrowed. In the example above, the collateral factor is 80%.

Specifically

$$
\text{collateral\_factor}=\frac{\text{loan\_value}}{\text{collateral\_value}}
$$

Collateral factor is expressed as a percentage. Generally, more volatile and less liquid assets require a _lower_ collateral factor. For example, for extremely volatile assets, the contract might require someone to deposit $1,000 of collateral to borrow $500 if the assets are subject to rapid price swings. This is a 50% collateral factor.

On the flip side, if assets are very liquid and stable, the contract might tolerate a very high collateral factor, say 90%.

The higher the collateral factor required, the more competitive the lending protocol because borrowers want to put down as little collateral as possible relative to the loan they are trying to take out.

The lower the collateral factor, the higher the margin of safety for the lender. The relationship between the collateral factor and margin of safety is illustrated in the following GIF.

![collateral factor and margin of safety](https://static.wixstatic.com/media/935a00_96e58f3957104b33aa23d85222288c25~mv2.gif)

## Loan to Value (LTV)

The loan to value is the **_current_** value of the loan as a percentage of the collateral value. For example, the required collateral factor may be 80%, but the value of the assets and collateral may fluctuate such that the ratio of the asset value to the collateral value becomes 70%.

## Liquidation factor

If the loan to value gets too close to 100%, there is a risk that the amount of money borrowed will exceed the value of the collateral causing the lender to lose out. When the collateral is worth less than the loan, the LTV exceeds 100% and we have **bad debt**  and the protocol risks becoming **insolvent** (unable to pay back the lenders).

**Example:** Alice took out an <span style="color:#008aff">$800</span> loan against her collateral asset worth <span style="color:Orange">$1000</span>. Alice would withdraw <span style="color:#8d28a4">$800 from</span> the protocol. The lender currently has a <span style="color:Green">buffer of $200</span>.

![LTV when alice borrows](https://static.wixstatic.com/media/935a00_5f733c562c8a46eb905a2a1805a88c26~mv2.png/v1/fill/w_740,h_106,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_5f733c562c8a46eb905a2a1805a88c26~mv2.png)

Now, say after 20 blocks, the price of the collateral asset went down <span style="color:Red">30%</span>. Now worth <span style="color:Orange">$700</span>.

![LTV after Alice's collateral drops in value](https://static.wixstatic.com/media/935a00_527558e4313d45d39d4602199864b1ed~mv2.png/v1/fill/w_740,h_126,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_527558e4313d45d39d4602199864b1ed~mv2.png)

Since Alice took out a $800 loan and her collateral asset is now worth $700 , she would have no motivation to pay it back.

The protocol which holds her asset has to pay the lenders back $800 but only has $700 worth of asset, incurring a “bad debt” of $100.

To prevent this from happening, assets have a liquidation factor — a threshold percentage at which the collateral can be forcibly taken from the borrower.

The liquidation factor is always a higher percentage than the collateral factor. If not, the borrower could get liquidated the moment they borrow a loan. For example, the collateralization factor might be 80% but the liquidation factor at 90%. Setting the liquidation factor too close to 100% increases the risk of bad debt as the collateral factor might exceed 100% at the time of liquidation.

## How LTV increases

There are three ways for the loan-to-value to increase and to approach the liquidation factor.

1.  The loan accrues [interest](https://www.rareskills.io/post/aave-interest-rate-model) and has a larger notional value

2.  The collateral falls in value

3.  The borrowed asset rises in value

**1) Example: the loan accrues interest and has a larger notional value**

A user deposits $1,000 of ETH and borrows $800 of USDC. The interest accumulates until the loan balance is $900 USDC. If the liquidation factor is 90%, it is subject to liquidation.

![loan accruing interest and getting liquidated](https://static.wixstatic.com/media/935a00_94589a848f3a4aab8313cdaa89bcb159~mv2.gif)

**2) Example: the collateral falls in value**

A user deposits $1,000 of ETH and borrows $800 of USDC. The smart contract has set a liquidation factor of 90%. If the ETH drops to $888.88 in value, then the liquidation will be triggered as the LTV will be 90%.

![loan getting liquidated due to collateral dropping in value](https://static.wixstatic.com/media/935a00_1f618e262897480a932048e5d7c93046~mv2.gif)

**3) Example: rising debt value**

A user deposits $1,000 ETH and borrows $800 USDC. USDC depegs and becomes worth $1.2 due to extreme market volatility. Now the notional value of the debt is $960 (800 * 1.2). Since the LTV is now 96%, it is subject to liquidation.

## Liquidation mechanism

The exact mechanism of the liquidation varies by protocol. In a simple case, the liquidator pays off the USDC loan of the borrower and receives a portion of the collateral at a discount.

Liquidation is usually permissionless. Anyone can call the function on the smart contract to trigger the liquidation.

### Compound V3’s mechanism

Compound V3 does not require the liquidator to pay off the USDC loan. Rather, it uses USDC that has not been lent out (un-utilized USDC) to pay off the loan. Then, the protocol becomes the owner of the collateral. This transaction is called “absorbing” the debt. Now the debt has been paid off from the reserves, and the protocol is in possession of excess collateral. Excess collateral is immediately put on sale at a discount to the oracle price.

A liquidator in Compound V3 will first call `absorb()` on the underwater loan to absorb the collateral into the protocol, and then immediately call `buyCollateral()` in the same transaction. To incentivize liquidators, collateral can usually be bought at a discount to current market prices.

There is no financial incentive for calling `absorb()` in Compound V3 (though governance could program it in). However, there is a financial incentive for buying collateral at a discount, so MEV liquidators are constantly hunting for opportunities to absorb debt and subsequently buy the newly available collateral.

**Example liquidation**

Compound V3 has $100,000 in USDC deposited by lenders. A borrower deposits $50,000 of collateral in LINK and borrows $25,000 USDC. Now $75,000 USDC is idle, sitting unused in Compound V3. The LINK value drops to $30,000 causing the collateral to go underwater and a liquidator calls `absorb()`. After `absorb()` is called, the balance in Compound V3 is still $75,000 (because the delinquent borrower holds $25,000). However, the protocol now takes possession of the $30,000 LINK the borrower deposited. The liquidator will purchase the $30,000 LINK for $28,000 USDC due to the discount. The 28,000 USDC is added to protocol’s 75,000. Now the protocol has $103,000 USDC and no collateral.

### Considerations for changing the liquidation factor while loans are active

If the protocol has a mechanism to change the liquidation factor, borrowers need to get sufficiently advanced warning. If the liquidation factor changes suddenly with no notice, then they will get unfairly liquidated.

## Lending protocols usually do not seize the entire collateral

In practice, lending protocols would not cause a 100% loss of collateral for the borrower in the example above. In practice, they would impose something like a 10% “liquidation penalty” and only give a portion of that to the liquidator.

**Example liquidation penalty**

Using the same values as the previous example, the borrower is liquidated when the LINK value has dropped to $30,000. The borrower borrowed 25,000 USDC. When the liquidator pays off the 25,000 USDC loan, the borrower is subject to a 10% liquidation penalty, meaning they lose out on 10% of the $30,000, or $3000. Another $25,000 is subtracted from the LINK to account for the unpaid loan. This leaves 30k - 3k - 25k = 2k in USDC that is credited to the borrower’s account.

## Storefront Price

Typically, the liquidation penalty is split between the liquidator and the protocol. The portion that goes to the liquidator is a ratio defined by the what Compound V3 calls the “storefront price.” In the example above, if the storefront price is 50%, then 1,500 USDC goes to the liquidator and 1,500 USDC goes to the protocol, since the liquidation penalty was 3,000 USDC.

## Danger of Small Loans

If the amount of collateral deposited is too low, then the liquidation penalty that goes to the liquidator might not pay for the gas of the transaction to liquidate the borrower. With no incentive to liquidate loans that are close to getting underwater, then the protocol will accumulate bad debt. To avoid these problems, lending protocols generally enforce that loans have a minimum size.

## Danger of Large Loans

Liquidating a large amount of collateral can lead to price impacts where the recovered amount is less than the borrowed amount.

**Example price impact during large liquidations**

A borrower has deposited $10 billion of ETH to a lending contract and has borrowed 9 billion dollars of USDC. The price of ETH drops to $9 billion. If liquidators try to sell $9 billion dollars of ETH on the open market, there is a realistic possibility this will drive the price down so much that they end up selling it for $8 billion. This results in a $1 billion loss to the protocol because the borrower has withdrawn $9 billion USDC but the liquidators have obtained $8 billion from the sale of ETH.

It isn’t important that the 10 billion dollars of ETH collateral is deposited by one borrower. If 10 borrowers with 1 billion dollars of ETH collateral each get liquidated simultaneously, the effect is the same.

### The Supply Cap in Compound V3

To avoid the issue described above, Compound V3 imposes a per-asset `supplyCap` of the maximum amount of collateral it will hold. This parameter is set by governance based on how much of the collateral can be liquidated without too much adverse price impact.

## Danger of Flash Crashes

Any time collateral is involved, the lender must assume the price will not fall too quickly. This is a risk that cannot be completely eliminated as flash crashes are always possible.

### How are risk parameters set?

It is not possible to 100% eliminate risk in DeFi lending. For example, under a very extreme situation where prices instantaneously crash 90%, then the lenders will lose out. There is always a tradeoff in risk-return.

The protocol could enforce a liquidation threshold of 10%, in which case the margin of safety is significant, but nobody would borrow from the protocol. Lending protocols have two conflicting goals to optimize for: capital efficiency and risk. The more conservative the risk factors, the less competitive the loan offerings are.

Although parameters are set by governance, the parameters are usually recommended by consulting agencies that specialize in financial models.

[Gauntlet](https://www.gauntlet.xyz/) currently suggests the risk parameters for Aave and Compound. They maintain [risk dashboards for these protocols](https://risk.gauntlet.xyz/).

Gauntlet attempts to predict the worst case liquidation scenarios at times of market stress and set the parameters such that Compound will not incur bad debt during all the liquidations. Factors they consider include asset volatility and correlation of asset prices during normal times and times of market stress. You can read here about their [risk methodology](https://medium.com/gauntlet-networks/improved-var-methodology-9f4f0c4cdb6f).

At the time of writing, Gauntlet renews their contract with Compound annually at a rate of [2 million dollars per year](https://www.comp.xyz/t/gauntlet-compound-renewal-2023/4638).

## Other vulnerabilities

There are other vulnerability vectors for lending contracts that we do not discuss here, as we are solely focused on topics related to collateral and liquidation. Please see our article on [smart contract security](https://www.rareskills.io/post/smart-contract-security) for other lending vulnerability vectors.

## Conclusion

Loans in DeFi must be overcollateralized. The ratio of the loan to collateral value that the protocol will accept to initiate a loan is the collateralization ratio. The liquidation ratio is the ratio of values at which the loan will be liquidated. Smart contracts cannot initiate transactions themselves, so they rely on liquidators to buy the collateral at a discount. Loans cannot be too small relative to gas prices or too high relative to available liquidity, otherwise there will be issues liquidating bad debt.

## Learn more with RareSkills

Please see our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) to learn more technical web3 topics.

*Originally Published December 30, 2023*
