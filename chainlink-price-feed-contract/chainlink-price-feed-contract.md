# How Chainlink Price Feeds Work

Chainlink price oracles are smart contracts with public view functions that return the price of a particular asset denominated in USD.

Off-chain nodes collect the prices from various sources like exchanges and write the price data to the smart contract.

Here is the smart contract for getting the price of ETH / USD: [https://etherscan.io/address/0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419/advanced#readContract](https://etherscan.io/address/0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419/advanced#readContract)

When we call the function `latestAnswer()` we get the price of Ether. When we query `decimals()` we get the number of decimals to interpret the answer with.

![chainlink latestAnswer() function return](https://static.wixstatic.com/media/935a00_7ceef569b4014c48b09ce40b993ab905~mv2.png/v1/fill/w_740,h_146,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_7ceef569b4014c48b09ce40b993ab905~mv2.png)

Therefore, the current price of Ether is $2053.05552675 according to the oracle (true at the time of writing).

If you just want an idea of how Chainlink oracles work, you can stop here — that’s all a price oracle is!

What follows is important implementation details if you plan on using them in a project.

We will use ETH / USD as a running example, but Chainlink supports many more asset prices.

## Using latestAnswer() is not recommended — use latestRoundData() instead

This function `latestAnswer()` does not tell us the last time the price updated. If price updates are delayed, the smart contract might make decisions based on outdated prices.

In the <span style="color:Green">green box</span> below, we see the same price we got from latestAnswer() and in the <span style="color:#008aff">blue box</span> we see when it was last updated as a Unix timestamp.

![latestRoundData() function return on Etherscan](https://static.wixstatic.com/media/935a00_be306db643fe4f89b18b590f3b24a006~mv2.png/v1/fill/w_740,h_430,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_be306db643fe4f89b18b590f3b24a006~mv2.png)

Smart contracts may want to set a threshold such that they use an alternative oracle or suspend critical decisions until the `updatedAt` field is sufficiently recent.

## Price Aggregation

It would be unsafe to rely on a single node or data source to obtain prices, so Chainlink price feeds have several whitelisted nodes that supply prices.

The [suppliers of the ETH / USD price feed](https://data.chain.link/ethereum/mainnet/crypto-usd/eth-usd) are screenshotted below.

The range of prices they supply at a given time can be seen in the small variation in reported price. Note that the “cents” portion of the price varies from 26 cents (top row) to 73 cents (bottom row).

![list of Chainlink price feed providers](https://static.wixstatic.com/media/935a00_9180889c6ed243d8a81d3ab4f51b1ece~mv2.png/v1/fill/w_740,h_586,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_9180889c6ed243d8a81d3ab4f51b1ece~mv2.png)

## transmit()

The off-chain prices enter the smart contract ecosystem via the [transmit function](https://etherscan.io/address/0xE62B71cf983019BFf55bC83B48601ce8419650CC#writeContract). The function takes a list of (sorted) prices and a list of signatures from the nodes. The price reported on the oracle is the median of the prices. Below we show the relevant line of code from Etherscan.

![median computation in the transmit() function](https://static.wixstatic.com/media/935a00_be0c1532d9fa413b931b36a1902abe8c~mv2.png/v1/fill/w_740,h_281,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_be0c1532d9fa413b931b36a1902abe8c~mv2.png)

## Smart contract Architecture

The reader may have noticed that the `latestRoundData()` function is not in the same contract as `transmit()`. There are three smart contracts at play:

1.  The [price feed contract](https://etherscan.io/address/0x5f4ec3df9cbd43714fe2740f5e3616155c5b8419)

2.  The [aggregator contract](https://etherscan.io/address/0xE62B71cf983019BFf55bC83B48601ce8419650CC)

3.  The [validator contract](https://etherscan.io/address/0x264BDDFD9D93D48d759FBDB0670bE1C6fDd50236)

## Price update transaction

During a price update, the signatures and prices of the nodes are batched together and sent to `transmit()` in the aggregator contract. The aggregator contract then calls the [validate function](https://etherscan.io/address/0x264BDDFD9D93D48d759FBDB0670bE1C6fDd50236/advanced#writeContract) in the validator contract. Subject to the rules there, the price update might be rejected. The [tenderly trace](https://dashboard.tenderly.co/tx/mainnet/0x536c5be15489ac91a283aaeaaadd9f7fdce178aafbfc47d939674f6b5ab8248e?trace=0) of such a transaction is screenshotted below. The <span style="color:#8d28a4">purple call codes</span> show the cross contract calls.

![tenderly trace of price update](https://static.wixstatic.com/media/935a00_2cc72b4f4f6a4d6e8ea4933677e472f4~mv2.png/v1/fill/w_740,h_142,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_2cc72b4f4f6a4d6e8ea4933677e472f4~mv2.png)

## Viewing the price

![aggregator address](https://static.wixstatic.com/media/935a00_305954dd6bc84bb1acbbb47ea7316b27~mv2.png/v1/fill/w_350,h_115,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_305954dd6bc84bb1acbbb47ea7316b27~mv2.png)

### Improving the gas efficiency of reading price oracles

Because viewing the price involves a cross-contract call, it is recommended to save 200 gas by “pre-warming” the aggregator call using an [access list transaction](https://www.rareskills.io/post/eip-2930-optional-access-list-ethereum). See an example of using an access list with a Chainlink price oracle in this [repo](https://github.com/RareSkills/access-list-benchmarks/tree/main/chainlink_oracle).

## Price update frequency

It isn’t practical to keep sending state-updating transactions to the blockchain every minute. Therefore, Chainlink updates the price under two circumstances:

-   When the "heartbeat" time passes (for ETH / USD this is one hour)
-   If the price changes by more than 0.5%

These parameters are highlighted in a screenshot of the chainlink ETH / USD dashboard below

![ETH / USD trigger parameters for price update](https://static.wixstatic.com/media/935a00_0208ee9e5dbf4341aec76feaf2af6eb0~mv2.png/v1/fill/w_350,h_554,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_0208ee9e5dbf4341aec76feaf2af6eb0~mv2.png)

## Security considerations

Much has been written about [smart contract security](https://www.rareskills.io/post/smart-contract-security) issues that arise out of using Chainlink oracles incorrectly, so we won’t repeat it here. We refer the reader to an [article](https://medium.com/cyfrin/chainlink-oracle-defi-attacks-93b6cb6541bf) by Dacian which lists those potential issues.

## Learn more with RareSkills

Please see our [solidity bootcamp](https://www.rareskills.io/solidity-bootcamp) to learn smart contract development at a deep level.

*Originally Published January 11, 2024*
