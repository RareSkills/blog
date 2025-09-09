# Bulkers in Compound V3

The bulker contracts in Compound V3 are multicall-like contracts for batching several transactions.

For example, if we wanted to supply Ether, LINK, and wBTC as collateral and borrow USDC against it in one transaction, we can do that.

We can also reduce the collateral holdings and withdraw a loan in one transaction as the screenshot below demonstrates. This of course assumes that we stay within the collateral factor limits.

![bulker example](https://static.wixstatic.com/media/935a00_40cc6de5a1fb4935940e6ed55107a3b3~mv2.png/v1/fill/w_666,h_691,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_40cc6de5a1fb4935940e6ed55107a3b3~mv2.png)

## invoke()

The bulker does not behave like a traditional multicall where it accepts a list of arbitrary calldata. Instead, it takes two arguments: a list of actions (of which there are 6 choices) and the arguments to supply to them. The function is shown below. Specifically, we can

-   supply an ERC-20 (`ACTION_SUPPLY_ASSET`)
-   supply ETH (`ACTION_SUPPLY_NATIVE_TOKEN`)
-   transfer an asset (`ACTION_TRANSFER_ASSET`), see our article on how [Compound V3 behaves like a rebasing ERC-20 token](https://www.rareskills.io/post/cusdc-v3-compound) to see how this works
-   withdraw an ERC-20 (`ACTION_WITHDRAW_ASSET`)
-   withdraw ETH (`ACTION_WITHDRAW_NATIVE_TOKEN`)
-   claim accumulated COMP rewards. The underlying function claimReward will interact with the [reward contract](https://github.com/compound-finance/comet/blob/main/contracts/CometRewards.sol) instead of the main lending contract (Comet.sol).


![invoke function](https://static.wixstatic.com/media/935a00_5344e6d6cf814662b7d3225f03e0dd85~mv2.png/v1/fill/w_666,h_551,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_5344e6d6cf814662b7d3225f03e0dd85~mv2.png)

Invoke will loop through the actions and call Comet (or the rewards contract) with the arguments supplied.

Although it could have been more gas efficient to put this code into the main contract, using msg.value in a loop, especially when invoking [delegatecall](https://www.rareskills.io/post/delegatecall), is not safe. See the practice problem at the end of this article.

Compound is careful to make sure msg.value is deducted rather than re-used, which could lead to double spending — see the <span style="color:#c1c146">yellow boxes</span>.

This arguably could have been [gas-optimized](https://www.rareskills.io/post/gas-optimization) by having the actions decided with a 1 byte indicator rather than a 32-byte ASCII string indicator.

## The bulkers are non-custodial

The bulker never holds the tokens of the user. Instead, the user gives approval to the bulker and the bulker supplies assets to Compound on behalf of the user. Compound does not assume that `msg.sender` is the depositor; doing so is generally not a good design pattern because it breaks composability — it prevents other contracts from acting on behalf of the user.

## Rescuing tokens

In the case where users accidentally transfer tokens to the bulker, an admin can remove the tokens. Note we have separate functions to remove ETH that is stuck and ERC-20 tokens that are stuck.

![sweepToken function](https://static.wixstatic.com/media/935a00_253fa5344162490ca147520d3eaa5a22~mv2.png/v1/fill/w_666,h_485,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_253fa5344162490ca147520d3eaa5a22~mv2.png)

## Handling non-standard ERC-20 tokens

One of the most common deviations from the ERC-20 token specification is not returning a boolean value. Several ERC-20 tokens do not return a boolean but revert in the failure case.

To handle both situations when dealing with ERC-20 assets, Compound implements the code below:

![handling non-standard erc20 tokens](https://static.wixstatic.com/media/935a00_09ba74d20e7f47fc86e8967deba37c5b~mv2.png/v1/fill/w_666,h_245,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_09ba74d20e7f47fc86e8967deba37c5b~mv2.png)

There is no return value of IERC20NonStandard — it will revert if dealing with a non-standard ERC-20 token. To handle the possibility that this is actually a standard ERC-20 token, the case where the return data size is 32 bytes will signify that the token did return a value. If the returned value is 1, it succeeded, and otherwise failed. The different scenarios are captured in the matrix below.

![An image of different scenario matrix](https://static.wixstatic.com/media/935a00_7a8c7c122a1a445791cab21cfa5a2e51~mv2.png/v1/fill/w_666,h_193,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_7a8c7c122a1a445791cab21cfa5a2e51~mv2.png)

Essentially, we succeed on the cases where the (token does not revert AND returns nothing) OR (token does not revert AND returns true). We fail on cases where (token reverts) OR (token does not revert AND returns false).

The [IERC20Nonstandard interface](https://github.com/compound-finance/comet/blob/main/contracts/IERC20NonStandard.sol) simply defines an ERC20 token that does not return any booleans.

![nonstandard erc20 interface](https://static.wixstatic.com/media/935a00_0eeb710ebd434d5090c72caafe1f50fb~mv2.png/v1/fill/w_666,h_269,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_0eeb710ebd434d5090c72caafe1f50fb~mv2.png)

## Practice problems

Bad things can happen when using `msg.value` inside a loop. See these two practice problems:

1.  [RareSkills Riddles MultiDelegateCall](https://github.com/RareSkills/solidity-riddles/blob/main/contracts/MultiDelegateCall.sol)

2.  [DamnVulnerableDefi Free Rider](https://www.damnvulnerabledefi.xyz/challenges/free-rider/)


## Learn More with RareSkills

Please see our [web3 bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) to learn more.

*Originally Published January 9, 2024*
