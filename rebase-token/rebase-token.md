# Coding a Solidity rebase token

A “rebase token” (sometimes “rebasing token”) is an ERC-20 token where the total supply, and the balances of token holders, can change without transfers, minting, or burning.

DeFi protocols often use rebasing tokens to track the amount of asset it owes to a depositor — including profit the protocol made. For example, if the protocol owes a depositor 10 ETH (including profit), the depositor’s ERC-20 balance for the rebasing token would be 10e18. If their deposit increased in value to 11 ETH, their balance would “rebase” to 11e18.

This article explains how to code rebasing token, as well as the logic behind the code.

We also cover potential security issues that may come up when creating a rebasing token.

## Example rebase token transactions

Consider the following example that illustrates rebasing tokens:

1. Alice deposits 100 ETH into a pool. The pool mints her 100 rbLP (rebasing LP token).
2. Alice could burn the 100 rbLP and get her 100 ETH back. The balance of her rbLP is the amount of ETH she can redeem from the pool.
3. But suppose the pool makes a profit of 10%, such as from lending fees. Her 100 rbLP balance will automatically “rebase” to 110 rbLP.
    1. That is, if we call `rbLP.balanceOf(alice)` right when she deposits, it would return 100 (with 18 decimals).
    2. After the profit-making, `rbLP.balanceOf(alice)` would return 110.
4. Now Bob deposits 100 ETH into the pool, after the pool makes a profit. The pool would mint him 100 rbLP. Alice however, has 110 rbLP since she was supplying liquidity before the pool made the 10% profit.

A rebase token attempts to keep the total supply of the rebase token equal to the total amount of ETH held by the pool. In reality, there may be slightly more ETH than the total supply of rebase tokens due to rounding errors — we will discuss this later.

The balance of rbLP token a user has is the amount of ETH they can redeem from the pool. Therefore, the balance of a user can be interpreted as their “share” of the total supply of the rebasing token (or equivalently, the amount of ETH held by the pool).

We will refer to the deposited asset as ETH for the rest of this article, but of course, it could be some other ERC-20 token.

## Designing a rebasing token

We will create a rebasing ERC-20 token. The ERC-20 token contract’s total supply is the amount of Ether held by the token (which means our rebasing token has 18 decimals). We will sometimes refer to the “token” as the “pool” interchangeably. One can think of this as a pool that implements the ERC-20 standard (but with rebasing) to track who is owed how much ETH.

*This design is heavily inspired by the [Lido stETH token](https://github.com/lidofinance/core/blob/master/contracts/0.4.24/StETH.sol).*

### balanceOf()

In a traditional ERC-20, the balance of a user is simply a number associated with the address in the `mapping(address => uint256)`.

In a rebasing ERC-20 token, the value held in the mapping represents the user’s fractional ownership of the pool. It’s best to think of the mapping as holding “shares.”

```solidity
mapping(address => uint256) internal _shareBalance;
```

The fraction of the total supply can be computed as `_shareBalance[user] / _totalShares`*, where* `_totalShares` is the sum of shares of all the users.

Suppose Alice owns 70% of the ETH in the pool, and Bob owns 30% of the ETH in the pool. A valid distribution of shares could be:

- Alice: 70 shares
- Bob: 30 shares

But we only care about shares as a ratio of ownership. The following share distribution would be valid in the same scenario:

- Alice: 35 shares
- Bob: 15 shares

The balanceOf the rebase token represents the amount of ETH the user can redeem. The amount of ETH a certain number of shares can redeem is:

$$
\frac{\texttt{num\_share}}{\texttt{total\_shares}}\times\texttt{ETH\_balance}
$$

Using the second example where Alice has 35 shares and Bob has 15 shares, we can see that Alice can redeem 70% of the ETH in the pool:

$$
\frac{35}{35+15}=0.7
$$

Translating the formula to Solidity we get:

```solidity
function balanceOf(address account) public view returns (uint256) {
    if (_totalShares == 0) {
        return 0;
    }
    return _shareBalance[account] * address(this).balance / _totalShares;
}
```

The variable `_totalShares` is updated during `mint()` and `burn()` when ETH is added or removed from the pool.

### mint()

During `mint()` the depositor adds Ether to the pool and mints an amount of shares that represents their percent ownership of the total percent of outstanding shares (the sum of all shares in existence, or the sum of all `_shareBalance[user]` for all users).

If they are the first minter, then the amount of shares minted is simply `msg.value`. Otherwise, they must maintain the ratio:

$$
\frac{\text{totalSharesPrevious}}{\text{totalBalancePrevious}}=\frac{\text{totalSharesPrevious}+\text{sharesToCreate}}{\text{totalBalancePrevious}+\texttt{msg.value}}
$$

This can be thought of as saying “the amount of balance a share can redeem is not changed by a mint.”

To solve for `sharesToCreate`, let’s shorten the variables as follows:

$$
\frac{s}{b}=\frac{s+c}{b+v}
$$

Where:

- $s$ is the previous shares
- $b$ is the previous balance
- $c$ is the shares to create and
- $v$ is `msg.value`.

We can extract $c$ with the following algebra:

$$
\begin{align*}
\frac{s}{b}=\frac{s+c}{b+v}\\
\frac{s\cdot(b+v)}{b}=s+c\\
\frac{sb+sv}{b}=s+c\\
s+\frac{sv}{b}=s+c\\
\frac{sv}{b}=c\\
\end{align*}
$$

Therefore, `sharesToCreate = sharesPrevious * msg.value / balancePrevious`. However, `balancePrevious` is not something we store, but it can be computed as `address(this).balance - msg.value`. Thus, our code for `mint()` is as follows (the following code is not fully secure yet!):

```solidity
function mint(address to) external payable {
	require(to != address(0), ERC20InvalidReceiver(to));
    uint256 sharesToCreate;
    if (_totalShares == 0) {
        sharesToCreate = msg.value;
    } else {
        sharesToCreate = msg.value * _totalShares / (address(this).balance - msg.value);
    }
    _totalShares += sharesToCreate;
    _shareBalance[to] += sharesToCreate;
    
    uint256 balance = sharesToCreate * address(this).balance / _totalShares;
    emit Transfer(address(0), to, balance);
}
```

Note that `address(this).balance` and `totalShares` increase by the same percentage amount. Therefore, the ratio

$$
\texttt{balanceOf\_user}=\frac{\texttt{address(this).balance}\times\texttt{shares[user]}}{\texttt{totalShares}}
$$

remains unchanged *mostly* during a mint. This is because computing `sharesToCreate` involved division, and the number of shares created for the minter may be slightly less than it should be, meaning their percentage ownership is slightly underrepresented. This means other users may experience a slight increase in percentage ownership.

However, if someone transfers ETH directly to the contract, or the contract makes a profit in ETH (i.e., not by someone minting), then the balance will increase, but the `_totalShares` would not. This would increase the value of the ratio in the formula above, causing the balance to rebase upward.

Be aware that an attacker can temporarily increase their balance by using a flashloan to mint the rebasing token. Thus, any business critical logic should not blindly rely on `balanceOf()` or `totalSupply()`.

It is also worth noting that in the formula:

```solidity
sharesToCreate = msg.value * _totalShares / (address(this).balance - msg.value);
```

it is possible for `sharesToCreate` to round down to zero if

- the pool has a significant ETH balance
- _totalShares is relatively low (i.e. the protocol has made a lot of profit)
- `msg.value` is small

Since it is possible for shares to round down to zero, the current implementation is vulnerable to a small deposit attack.

Specifically:

1. The attacker mints 1 wei
2. The attacker frontruns the victim’s mint of 1 ether by donating 100 ether to the pool.

In step 2, the `sharesToCreate` will be computed as:

$$
\underbrace{10^{18}}_\text{msg.value}\cdot\underbrace{1}_\text{total shares}/(\underbrace{(101\cdot10^{18}+1)}_\text{address(this).balance}-\underbrace{10^{18}}_\text{msg.value})
$$

This will round down to zero since the denominator is larger than the numerator. Now the victim has deposited 1 ether, but was minted nothing. The attacker owns all the shares, and thus has taken control of the victim’s deposit.

Thus, our rebase token must implement some kind of slippage protection.

We could create a “minimumShares” parameter, but this leaks the abstraction of shares to the user. In other words, integrators now have to think about “shares” as a separate value from “balances.”

An alternative safety measure that doesn’t require knowledge of shares is to check that the ratio of `sharesToCreate / _totalShares` is close to `msg.value / address(this).balance`. If `sharesToCreate` rounded down too much, then the ratio `sharesToCreate / _totalShares` will be a lot less than the ratio of the ether deposited relative to the total balance.

Since `sharesToCreate` slightly rounds down, we check that:

`sharesToCreate / _totalShares >= slippage * msg.value / address(this).balance`

where slippage is a value like 0.999 if the slippage is desired to be 0.1%. Of course, we cannot express 0.999 in Solidity, so we could use basis points instead (1 basis point is 0.01%, 10,000 basis points is 100%.). This leads to the following formula:

```solidity
// slippageBp is basis points, so 9900 means we tolerate a 1% slippage
sharesToCreate / totalShares >= slippageBp / 10_000 * msg.value / address(this).balance
```

To eliminate the fractions which will round down to zero, we multiply both sides of the inequality by `_totalShares * 10_000 * address(this).balance`. This gives us

```solidity
sharesToCreate * 10_000 * address(this).balance >= slippageBp * msg.value * _totalShares
```

Thus, our mint function can be updated as follows:

```solidity
function mint(address to, uint256 slippageBp) external payable {
		require(to != address(0), ERC20InvalidReceiver(to));
    uint256 sharesToCreate;
    if (_totalShares == 0) {
        sharesToCreate = msg.value;
    } else {
        sharesToCreate = msg.value * _totalShares / (address(this).balance - msg.value);
    }
    _totalShares += sharesToCreate;
    _shareBalance[to] += sharesToCreate;
    
    require(sharesToCreate * 10_000 * address(this).balance >= slippageBp * msg.value * _totalShares, "slippage");
    
    emit Transfer(address(0), to, msg.value);
}
```

There is room for gas optimization by not reading `_totalShares` and `_shareBalance[to]` from storage right after writing them, but we do not show this optimization for simplicity.

### totalSupply()

As stated earlier, the total supply of the rebasing token is ETH held by the pool:

```solidity
function totalSupply() public view returns (uint256) {
	return address(this).balance;
}
```

### Converting a token amount (balance) to shares

At this stage, it is useful to introduce a helper function `amountToShares`. It is the inverse function of the formula `balanceOf()` uses. Suppose a user wants to burn (or transfer) their entire balance. How many shares does that correspond to?

To compute this, we solve the `balanceOf` equation for `_shareBalance[user]`:

$$
\texttt{balance}=\frac{\texttt{address(this).balance}\times\texttt{shares}}{\texttt{totalShares}}
$$

After multiplying both sides by $\texttt{totalShares}$ and dividing by $\texttt{address(this).balance}$ we get:

$$
\texttt{shares}=\frac{\texttt{balance}\times\texttt{totalShares}}{\texttt{address(this).balance}}
$$

Thus, to convert a balance to shares, we use the following function:

```solidity
function _amountToShares(uint256 amount) internal view returns (uint256) {
    if (address(this).balance == 0) {
        return 0;
    }
    return amount * _totalShares / address(this).balance;
}
```

Now, when a user specifies an “amount” of underlying token (in our case, ETH) that they want to burn, we can straightforwardly convert it to shares.

### burn()

The argument to burn is the balance they are looking to burn, not the shares. During a burn, we convert the amount burned to the amount of shares, then deduct that from the shares of the user and totalShares.

```solidity
function burn(address from, uint256 amount) external {
    _spendAllowanceOrBlock(from, msg.sender, amount);
    uint256 shares = _amountToShares(amount);
    require(shares > 0, "zero shares");
    _shareBalance[from] -= shares;
    _totalShares -= shares;

    (bool ok,) = from.call{value: amount}("");
    require(ok, ERC20InvalidReceiver(from));

    emit Transfer(from, address(0), amount);
}
```

Again, the balance of a rebasing token is the amount of ETH they can withdraw. Therefore, the `amount` parameter is precisely the amount of ETH transferred to them at the end of `burn()`.

It is not necessary to check the balance of the user before deducting it, as Solidity (version 0.8.0 or later) revert on underflow.

The function `_spendAllowanceOrBlock()` we will revisit in a later section.

The `require(shares > 0, "zero shares");` is to prevent Ether from being transferred out of the contract if `shares` rounds down to 0. Recall that `_amountToShares` is computed as `amount * _totalShares / address(this).balance;`. If `amount * _totalShares` is less than `address(this).balance` then `shares` rounds to 0. Neither `_shareBalance[from] -= shares;` nor `_totalShares -= shares;` will revert due to underflow, so the caller could withdraw `amount` from the contract without that require statement.

### transfer() and transferFrom()

Transfer and transferFrom are both similar to burn, except instead of destroying the shares and sending the ETH, the shares are credited to another account and no ETH is transferred:

```solidity
function transferFrom(address from, address to, uint256 amount) public returns (bool) {
    require(to != address(0), ERC20InvalidReceiver(to));
    _spendAllowanceOrBlock(from, msg.sender, amount);
    uint256 shareTransfer = _amountToShares(amount);
    _shareBalance[from] -= shareTransfer;
    _shareBalance[to] += shareTransfer; 

    emit Transfer(from, to, amount);
    return true;
}
```

Because `amountToShares` carries out the computation `amount * _totalShares / address(this).balance` the amount of shares transferred might be rounded down.

**Because division rounds down, this means that the balance received by `to` might be very slightly less than `amount`.** 

This is an issue to be aware of in general for rebasing tokens — see the [Lido documentation on this issue for how stETH handles it](https://docs.lido.fi/guides/lido-tokens-integration-guide/#1-2-wei-corner-case).

### Rounding issue in burn

As a corollary, it is possible for a user to burn their entire balance but be left with a small balance because the computed number of shares to burn rounded down from the actual number of shares they own. Thus, we should not assume that the share balance goes to zero when the entire balance is burned.

### Allowance and Approve

There is no “correct” way to implement allowance and approve for rebasing tokens, since rebasing ERC-20 tokens don’t have a standard that dictates how they should behave.

However, most rebasing tokens use an allowance mechanism that is similar to a regular ERC-20 token — but the allowance does not rebase.

The drawback to this mechanism is that if Alice approves Bob for her entire balance, but there is a rebase before Bob transfers from Alice, then Bob cannot withdraw her entire balance.

The rebasing token Compound Finance uses solves this by only allowing “all or nothing” approvals. When calling `approve` the amount can only be 0 or `type(uint256).max` — but this can break integrations with protocols that specify an allowance for the amount they intend to transfer. The allowance and approval for AAVE and stETH (Lido) rebasing token on the other hand behave like a normal ERC-20 and do not correct for rebasing.

Therefore, the approval logic for our token is very similar to OpenZeppelin’s. We implement the function `_spendAllowanceOrBlock()` which, as the name suggests, spends the spender’s allowance and reverts if the allowance is not sufficient. In our implementation, we do not spend the allowance if `msg.sender == spender` and we do not deduct the allowance if the allowance is `type(uint256).max`.

We show it in the complete implementation in the next section.

## A complete Rebasing ERC-20

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {IERC20Errors} from "@openzeppelin/contracts/interfaces/draft-IERC6093.sol";
import {IERC20} from "@openzeppelin/contracts/interfaces/IERC20.sol";

contract RebasingERC20 is IERC20Errors, IERC20 {
    uint256 internal _totalShares;
    mapping(address => uint256) public _shareBalance;
    mapping(address owner => mapping(address spender => uint256 allowance)) public allowance;

    receive() external payable {}

    function mint(address to, uint256 slippageBp) external payable {
        require(to != address(0), ERC20InvalidReceiver(to));
        require(msg.value > 0);

        uint256 sharesToCreate;
        if (_totalShares == 0) {
            sharesToCreate = msg.value;
        } else {
            uint256 prevBalance = address(this).balance - msg.value;
            sharesToCreate = msg.value * _totalShares / prevBalance;
            require(sharesToCreate > 0);
        }
        _totalShares += sharesToCreate;
        _shareBalance[to] += sharesToCreate;
        require(sharesToCreate * 10_000 * address(this).balance >= slippageBp * msg.value * _totalShares, "slippage");

        uint256 balance = sharesToCreate * address(this).balance / _totalShares;
        emit Transfer(address(0), to, balance);
    }

    function burn(address from, uint256 amount) external {
        _spendAllowanceOrBlock(from, msg.sender, amount);
        uint256 shares = _amountToShares(amount);
        require(shares > 0);
        _shareBalance[from] -= shares;
        _totalShares -= shares;

        (bool ok,) = from.call{value: amount}("");
        require(ok, ERC20InvalidReceiver(from));

        emit Transfer(from, address(0), amount);
    }

    function _amountToShares(uint256 amount) public view returns (uint256) {
        if (address(this).balance == 0) {
            return 0;
        }
        return amount * _totalShares / address(this).balance;
    }

    function totalSupply() public view returns (uint256) {
        return address(this).balance;
    }

    function balanceOf(address account) public view returns (uint256) {
        if (_totalShares == 0) {
            return 0;
        }
        return _shareBalance[account] * address(this).balance / _totalShares;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        transferFrom(msg.sender, to, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) public returns (bool) {
        require(to != address(0), ERC20InvalidReceiver(to));
        _spendAllowanceOrBlock(from, msg.sender, amount);
        uint256 shareTransfer = _amountToShares(amount);
        _shareBalance[from] -= shareTransfer;
        _shareBalance[to] += shareTransfer;

        emit Transfer(from, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function _spendAllowanceOrBlock(address owner, address spender, uint256 amount) internal {
        if (owner != msg.sender && allowance[owner][spender] != type(uint256).max) {
            uint256 currentAllowance = allowance[owner][spender];
            require(currentAllowance >= amount, ERC20InsufficientAllowance(spender, currentAllowance, amount));
            allowance[owner][spender] = currentAllowance - amount;
        }
    }
}
```

Note that `name()`, `symbol()`, and `decimals()` have not been implemented in the code above.

## Some final notes

If someone transfers ETH to the contract after someone has already minted the rebasing token, the balance of the minter will rebase upward.

However, if someone transfers ETH to the contract before anyone mints, then the first minter will gain control of that ETH, and their balance will be equal to all the ETH in the pool since they own all the outstanding shares.

Some protocols cache the balance of ERC-20 tokens for gas efficiency — but this could break their logic if the token rebases.

Many protocols do not rebase automatically like our example above. Instead, they rebase daily or some other periodic interval. This may be necessary if the asset is not held in the contract (e.g. it is staked in Ethereum validators) or if the value of the assets depend on oracles.

When interacting with a rebasing token, the `amount` specified in the transfer might not equal the change in balance due to rounding of the shares. Therefore, it is better to use the following logic to determine the true amount deposited:

```solidity
uint256 balanceBefore = token.balanceOf(address(this));
token.transferFrom(sender, address(this), amount);
uint256 trueTransferAmount = token.balanceOf(address(this)) - balanceBefore;
```

[Ampleforth](https://www.ampleforth.org) and [OlympusDao’s sOHM token](https://docs.olympusdao.finance/main/contracts-old/tokens/#sohm) are two other notable rebase tokens. Ampleforth uses rebase tokens to dynamically peg the value of the rebase token to another asset. To increase the value of the token, it rebases downward (causing it to be more scarce), and when it needs to decrease the value of the token, it rebases upward to cause inflation.

*We would like to thank MerlinBoii from [Pashov Audit Group](https://www.pashov.net) and [deadrosesxyz](https://x.com/deadrosesxyz) for suggested revisions to an earlier versions of this article. We would like to thank* [ChainLight](https://chainlight.io) *for auditing the reference contract at the end of the article and identifying a serious vulnerability in an earlier implementation ([audit report](https://gist.github.com/this-is-chainlight/268e106904997e21ce664c1fdb0896b9)).*