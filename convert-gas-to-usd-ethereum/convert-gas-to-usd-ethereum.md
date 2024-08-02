# Convert gas to USD (Ethereum)

Understanding gas cost can be tricky because there are three components at play: the gas price, the price of ether, and the units of gas.

The "gas price" that you might see on a website like [etherscan.io/gastracker](https://etherscan.io/gastracker), which fluctuates with network demand.


The price of Ether. Obviously, the more expensive ether is, the higher the gas cost.


Finally, "gas" which is a unit that tells you how expensive the transaction is. Transferring Ether from one account to another costs 21,000 gas. Interactions with smart contracts cost more.


## [Calculator](https://www.rareskills.io/ethereum-gas-price-calculator)

You can use this handy [calculator](https://www.rareskills.io/ethereum-gas-price-calculator) if you just want to get a conversion done.

## Gas Price

The "gas price" is usually priced in units of Gwei. The values fluctuate between 15 during periods of low use to over 100 gwei during periods of high use. What exactly is a "gwei" in this context?

Ethereum, as a cryptocurrency, can be subdivided into decimal places of 10^-18. Each of these tiny units is called one "wei." One billion wei, or one giga-wei, is one billionth of an Ether, or 1 billion wei. (Multiplying 10^-18 by one billion is 10^-9, or one billionth).

"Gwei" is a handy unit of measure because gas prices happen to fluctuate around one billionth of an ether per unit of gas.

The gas price is how much units of **ether** (not dollars) you need to pay per unit of gas. The dollar cost comes from converting ether to dollars.

## Gas

Not to be confused with "gas price" which fluctuates, the gas cost is fixed per transaction. The more complex the transaction, the higher the gas cost. Ethereum transfers, for example, always cost 21,000 gas. Transferring ERC20 tokens usually costs approximately 50,000 gas.

Gas can be conceptualized as a "unit of computation" or very roughly, one clock cycle on a hypothetical CPU.

## Conversion

(21,000 gas) $\times$ (price of gas in gwei) $\times$ (price of ether in dollars) / (1 billion to normalize the gwei)

At the time of writing, Ether is about $1,600, the price of gas in gwei is 22 according to [etherscan](https://etherscan.io/gastracker), and an eth transfer requires 21,000 gas. Multiply that together and the gas cost of transferring Ether in dollars is 74 cents.

### How the units cancel each other out

21,000 gas x (unit of ether / gas) $\times$ (dollars / unit of ether). 

The "gas" terms cancel out, and the "unit of ether" cancel out, so our final transaction is priced in dollars.

## Why gas prices fluctuate

Ethereum limits each block to only use 30 million gas. Divide that by 21,000, and that means up to 1,428 Eth transfers can be done in one block. If more than 1,428 people want to make a transfer, they'll have to wait or pay a higher gas price. Remember, they can't change the 21,000 cost of a transfer, or the price of Ether (directly). The gas price is discretionary, transactors can pay as high a price as they chose, but for their transaction to be included, they must pay higher than the other transactors.

Because block space is limited, there is a supply and demand phenomenon at play. If more people want to send transactions than Ethereum has space for, then the most motivated buyer must outbid the other transactors.

## Gas Price Seasonality

These usage patterns are highly seasonal. Much of the trading happens in America daytime, so during their daytime hours, prices are usually higher. This [heatmap](https://ethereumprice.org/gas/) from [ethereumgasprice.org](http://ethereumgasprice.org) is very handy.

Here is a snapshot taking during the writing of this article. The local timezone is in Asia time, so during Asia's night time (United States day time), the gas price is higher. Also note that the weekend sees less usage.

![Ethereum gas price heatmap](https://static.wixstatic.com/media/935a00_5a32fa8e5cbf40a79cf698b6de0dfb50~mv2.png/v1/fill/w_666,h_276,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_5a32fa8e5cbf40a79cf698b6de0dfb50~mv2.png)

## Gas Guzzlers

Scroll down on the [etherscan gas page](https://etherscan.io/gasTracker#gasguzzler), and you'll see the smart contracts that cause the most gas consumption. The pattern of gas consumption follows the Pareto principle (80 20 rule), 80% of the gas consumption comes from 20% of the ecosystem.

At the time of writing, transaction fees associated with Uniswap consumes 10% of the network.

Ethereum burns most of the transaction fee (the miner receives a small portion), so this indicates that Uniswap alone has resulted in over 400 Ether being deleted over the past 24 hours.

![](https://static.wixstatic.com/media/935a00_927b4ff5d99b46afa6bfd1753963d83d~mv2.png/v1/fill/w_666,h_544,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_927b4ff5d99b46afa6bfd1753963d83d~mv2.png)

## Learn more

Our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp) teaches gas optimization and pricing very thoroughly.

*Originally Published March 7, 2023*