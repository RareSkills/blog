# Uniswap V2 Architecture: An Introduction to Automated Market Makers

Uniswap is a DeFi app that enables traders to swap one token for another in a trustless manner. It was one of the early automated market makers for trading (though not the first).

Automated market makers are an alternative to an [order book](https://www.investopedia.com/terms/o/order-book.asp), which the reader is assumed to already be familiar with.

## How AMMs work

An automated market maker holds two tokens (token X and token Y) in the pool (a smart contract). It allows anyone to withdraw token X from the pool, but they must deposit an amount of token Y such that the "total" of assets in the pool does not decrease, where we consider the "total" to be the product of the amounts of the two assets.

$$
xy \le x'y'
$$

Here $x'$ and $y'$ are the token balances of the pool after the trade and $x$ and $y$ token balances of the pool before the trade.

This guarantees that the pool's asset holdings can only stay the same or increase.
Most pools enforce some kind of a fee. Not only should the product of the balances increase, but it should increase by at least a certain amount to account for a fee.

Assets are provided to the pool by *liquidity providers*, who receive so-called LP tokens to represent their share of the pool. Liquidity provider balances are tracked in a manner similar to how [ERC 4626](https://rareskills.io/post/erc4626) works. The difference between an AMM and ERC 4626 is that ERC 4626 only supports one asset but an AMM has two tokens. Just like a vault, the liquidity providers' share of the pool stays the same, but the product $xy$ gets larger, so their slice is larger.

## Advantages of AMMs

### AMMs do not have a bid-ask spread

In an AMM, price discovery is automatic. It's determined by the ratio of assets in the pool. Specifically, if we have token $x$ and token $y$, price is determined as follows:

$$
\text{price}(x) = \frac {\text{poolHoldings }y}{\text{poolHoldings }x}
$$

And vice-versa for $y$. Specifically, the more of asset $x$ that is put into the pool, the more "abundant" it is, and the price of $x$ goes down.

There is no need to wait for a suitable "bid" or "ask" order to show up. It always exists.
If there is a mismatch between the price in an AMM and another exchange, then a trader will arbitrage the difference, bringing the prices back into balance.

We should emphasize that this is the "spot" or "marginal" price. If you buy any amount of $x$, the actual price you pay will be worse than the result of this calculation.

### AMMs doubled as an oracle

Since the price of the assets is automatically determined, other smart contracts can use an AMM as a price oracle. However, AMM prices can be manipulated with flash loans, so safeguards need to be put in place when using AMMs in this manner. Nonetheless, it is valuable that price data is provided for free.

### AMMs are highly gas efficient compared to order books

Order books requires a significant amount of bookkeeping (no pun intended). An AMM only needs to hold two tokens and transfer them according to simple rules. This makes them more efficient to implement.

## Disadvantages of AMMs

There are two major drawbacks to automated market makers: 1) the price always moves and 2) impermanent loss for liquidity providers.

### Even small orders move the price in AMMs

If you place an order to buy 100 shares of Apple, your order will not cause the price to move because there are thousands of shares available for sale at the price you specify. This is not the case with an automated market maker. Every trade, no matter how small, moves the price.

This has two implications. A buy or sell order will generally encounter more slippage than in an order book model, and the mechanism of swapping invites sandwich attacks.

### Sandwich attacks are largely unavoidable in AMMs

Since every order is going to move the price, MEV (Maximal Extractable Value) traders will wait for a sufficiently large buy order to come in, then place a buy order right before the victim's order and a sell order right after it. The leading buy order will drive up the price for the original trader, which gives them worse execution. It's called a sandwich attack, since the victim's trade is "sandwiched" between the attackers.

1) Attacker's first buy (front run): drives up price for victim
2) Victim's buy: drive up price even further
3) Attacker's sell: sell the first buy at a profit

### Liquidity providers don't have control over the price their assets are sold at

For reasons we will discuss later, liquidity providers can only provide assets proportional to the current ratio of tokens in the pool. For example, if there are 100 token $x$ and 200 token $y$, the new liquidity provider must provide twice as many token $y$ as $x$.

In a traditional order book, a market maker can place limit orders at levels they believe reflect a desirable bid or ask (For example, place a bid order below the current market price or place a sell order above the current market price), but this is not possible with an automated market maker. Remember that Automated Market Makers use a formula to set prices based on the asset ratios in the pool, as a result market makers cannot set specific prices at which they wish to sell their assets.

### Liquidity providers for AMMs may suffer from impermanent Loss

Let's say in a hypothetical scenario Ether starts at \$10 and becomes worth \$1,000 later.

If someone had a portfolio of 1 Ether and 10 USD, then their portfolio starts at \$20 and ends at \$1010 (1 ETH + \$10). Their profit is \$990 total.

If they kept the money in an AMM, then they would have missed out on most of the gains. The AMM would have 0.1 ETH and 100 USD after the price change. This correctly prices ETH at \$1,000, but the net value of the pool is less than \$990. Here is what the pool holdings will look like before and after the price change

$$
\begin{split}
(\text{ETH\_before})(\text{USD\_before}) & \le (\text{ETH\_after})(\text{USD\_after})\\
(1 \text{ ETH})(10 \text{ USD}) & \le (0.1 \text{ ETH})(100 \text{ USD})\\
10 & \le 10
\end{split}
$$

Although the amount of stablecoins went up 10x, the amount of Ether went down. The net result is that the value of our assets, when in the pool, increased less than if we'd held the assets separately.

Here is a table showing the relative performance of holding ETH and USD in a pool vs just holding them.

<table>
<tr><th></th><th>Ether Pool Balance</th><th>Stablecoin Pool Balance</th><th>ETH × Stablecoin</th><th>\$ value of 1 ETH</th><th>Value of assets if providing liquidity</th><th>Value of just holding</th><th></th></tr>
<tr><td>Before</td><td>1</td><td>10</td><td>10</td><td>10</td><td>$\$20 \; (\$10 \text{ ETH} + 10 \text{ USD})$</td><td>$\$20 \; (\$10 \times 1 \text{ ETH} + 10 \text{ USD})$</td><td></td></tr>
<tr><td>After</td><td>0.1</td><td>100</td><td>10</td><td>1000</td><td>$\$200 \; (\$100 \text{ ETH} + 100 \text{ USD})$</td><td>$\$1010 \;(\$1000 \times 1 \text{ ETH} + 10 \text{ USD})$</td><td></td></tr>
<tr><td>Gain</td><td></td><td></td><td></td><td></td><td>$\$180$</td><td>$\$990$</td><td></td></tr>
<tr><td></td><td></td><td></td><td></td><td></td><td></td><td></td><td></td></tr>
<tr></tr>
<tr></tr>
</table>

The missed out on gains are called "impermanent loss." In the table above, the impermanent loss is $\$810 = (\$990 - \$180)$.

## Architecture of Uniswap V2

The architecture of Uniswap V2 is surprisingly simple. At its core is the **UniswapV2Pair** contract that holds two ERC 20 tokens that traders can swap against, or liquidity providers can provide liquidity for. Every different possible **UniswapV2Pair** has a different UniswapV2Pair contract to manage it. If the desired **UniswapV2Pair** contract does not exist, a new one can be permissionlessly created from the **UniswapV2Factory** contract. **UniswapV2Pair** contracts are also ERC 20 tokens (they inherit from ERC 20), and that token is used to track deposits similar to how ERC 4626 works.

Although advanced traders or smart contracts can interact directly with a **pair** contract, most users will interact with a **pair** through a **router** contract, which has several convenience functions such as trading between **pairs** in one transaction to create a "synthetic" pair if it doesn't exist.

That's it! There's really only three smart contracts at play in the Uniswap V2 system.

**Factory:** [github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Factory.sol](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Factory.sol)

**Pair:** (which inherits ERC20): [github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol)

**Router:** [github.com/Uniswap/v2-periphery/tree/master/contracts](https://github.com/Uniswap/v2-periphery/tree/master/contracts)

### The core - periphery pattern

Observe that the router contract above is in a repository called "v2 periphery" and the pair is in the "v2 core" repository. Uniswap V2 follows the "core / periphery" design pattern where the most essential logic is held in the core while the "optional" logic is held in the periphery.

The intent behind this is to have the core hold as little code as possible, which reduces the possibility of bugs in the core business logic.

### How to locate a pool, given two token addresses

Instead of accessing a mapping from token pairs to pool address, smart contracts calculate the address of the pool by predicting the create2 address as a function of the token addresses and the factory address. Since there is no storage access, this is very gas efficient. Below is the helper function provided by UniswapV2Library for calculating the address of the Pair contract.

```solidity
// calculates the CREATE2 address for a pair without making any external calls
function pairFor(address factory, address tokenA, address tokenB) internal pure returns (address pair) {
    (address token0, address token1) = sortTokens(tokenA, tokenB);
    pair = address(uint(keccak256(abi.encodePacked(
            hex'ff',
            factory,
            keccak256(abi.encodePacked(token0, token1)),
            hex'96e8ac4277198f8fbbf785487aa39f430f63b76db002cb326e37da348845f' // init code hash
    ))));
}
```

### Why not use clones

The [EIP 1167 clone](https://rareskills.io/post/eip-1167-minimal-proxy-standard-with-initialization-clone-pattern) pattern is used to create a collection of similar contracts, so why not use that here? Although the deployment would be cheaper, it would introduce an extra 2,600 gas per transaction due to the [delegatecall](https://rareskills.io/post/delegatecall). Since pools are intended to be used frequently, the cost savings from deployment would eventually be lost after a few hundred transactions, so it is worth deploying a pool as a new contract.

## Practice Problems

It's easy to compute the amount of tokens required for a swap incorrectly, which could lead to the pool getting drained. Practice this with the following security challenge: [Ethernaut 22 Dex](https://ethernaut.openzeppelin.com/level/22)

## Learn More with RareSkills

This article is part of a series. Please see the [Uniswap V2 Book](https://rareskills.io/uniswap-v2-book) for the rest. Also see our [Blockchain Bootcamps](https://rareskills.io/web3-blockchain-bootcamps) for our other courses.

*Originally Published Nov 15, 2023*
