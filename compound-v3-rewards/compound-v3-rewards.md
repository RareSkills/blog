# How Compound V3 Allocates COMP Rewards

Compound issues rewards in COMP tokens to lenders and borrowers in proportion to their share of a market’s lending and borrowing.

The algorithm is extremely similar to the [MasterChef Staking Algorithm](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=&ved=2ahUKEwjOoYf13uWCAxUT1zgGHW39CBc4HhAWegQIChAB&url=https%3A%2F%2Fwww.rareskills.io%2Fpost%2Fstaking-algorithm&usg=AOvVaw1rOJ_0I4WKVtnWoSAwHbE1&opi=89978449), so the reader should familiarize themselves with that first.

## High level overview of Compound V3 rewards

Similar to MasterChef, the Compound V3 reward contract tracks how much one hypothetical “staked” USDC has earned since the beginning of time. Here “staking” can mean either borrowing or lending — both activities are rewarded.

Analogous to Compound V3’s `baseSupplyIndex` or MasterChef’s `rewardPerTokenAcc`, Compound rewards has `trackingSupplyIndex` and `trackingBorrowIndex` which track rewards for one USDC (lent or borrowed) since the beginning of time.

Like MasterChef, the amount of rewards a single USDC collects is “diluted” when more USDC is “staked” and vice versa.

Unlike MasterChef, the reward per unit of time is set with the immutable variables `baseSupplyTrackingSupplySpeed` and `baseTrackingBorrowSpeed`. Governance has an option to “rescale” the rewards that are distributed.

Users claim rewards from [Comet Rewards](https://etherscan.io/address/0x1b0e765f6224c21223aea2af16c1c46e38885a40#code) a separate contract from the main Comet lending contract.

Compound does not issue rewards if the total amount borrowed or total amount lent are below certain thresholds for reasons we will discuss later.

Like MasterChef, rewards are not automatically distributed, they must be claimed in a separate transaction. The screenshot below shows the frontend for claiming accumulated COMP tokens, with the action to claim the tokens in the <span style="color:#c1c146">yellow circle</span>.

![claim COMP UI](https://static.wixstatic.com/media/935a00_d692eb35b4064a389c55acb3ea24ed6b~mv2.png/v1/fill/w_666,h_395,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_d692eb35b4064a389c55acb3ea24ed6b~mv2.png)

### The Comet Rewards Contract Requires Occasional Topping Up

The Comet Rewards contract does not mint COMP tokens, it relies on Governance transferring tokens to it. There are a total supply of 10 million COMP tokens in circulation, and all of them have already been minted. A significant amount of the supply is held by governance. Periodically, COMP tokens are transferred from the governance treasury to the reward contract. You can see the following governance transactions that “top up” the reward contract.

[https://compound.finance/governance/proposals/194](https://compound.finance/governance/proposals/194) (Nov 21, 2023)

[https://compound.finance/governance/proposals/164](https://compound.finance/governance/proposals/164) (June 29, 2023)

The mainnet address for the rewards contract is

[0x1B0e765F6224C21223AeA2af16c1C46E38885a40](https://etherscan.io/address/0x1B0e765F6224C21223AeA2af16c1C46E38885a40#code)

Because there is a fixed supply, the COMP rewards for ecosystem participation cannot continue indefinitely unless governance buys COMP tokens on the open market.

In the Etherscan screenshot below we see most of the transactions with the contract are to claim COMP tokens (<span style="color:#008aff">blue box</span>), and that the contract is currently holding ~73,000 COMP tokens (<span style="color:#008aff">blue arrow</span>).

![Etherscan compound reward contract](https://static.wixstatic.com/media/935a00_37a8a9ea1505419ebbaa7567872fc3bb~mv2.png/v1/fill/w_666,h_458,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_37a8a9ea1505419ebbaa7567872fc3bb~mv2.png)

## trackingSupplyIndex and trackingBorrowIndex behave like rewardPerTokenAcc

The plot below should be familiar from MasterChef. The more USDC that is “staked” the less reward each token receives because only a fixed amount is issued each period (determined by `trackingSupplyIndex` and `trackingBorrowIndex`).

One noteworthy variation is that if the amount of USDC supplied or borrowed (<span style="color:Pink">pink line</span>) is below the `baseMinForRewards` (<span style="color:Red">red text and dashed line</span>) threshold, then USDC does not accumulate rewards, and the <span style="color:Green">trackingSupplyIndex</span> (or trackingBorrowIndex) does not increase for that state update.

![trackingSupplyIndex growth](https://static.wixstatic.com/media/935a00_96c8e18abb454d1fb310acdb600bfad1~mv2.jpg/v1/fill/w_666,h_499,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_96c8e18abb454d1fb310acdb600bfad1~mv2.jpg)

These variables are not public, but can be retrieved via the `totalsBasic()` public function in CometExt. Since CometExt is a separate contract that Comet issues delegatecalls to, we cannot retrieve the values via Etherscan. Instead we use cast from [Foundry](https://www.rareskills.io/post/foundry-testing-solidity) to retrieve them, as the screenshot below shows.

![totalsBasic() query in foundry cast](https://static.wixstatic.com/media/935a00_c6d6f5cbbc5c4614a8f8045694fd849b~mv2.png/v1/fill/w_666,h_124,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_c6d6f5cbbc5c4614a8f8045694fd849b~mv2.png)

## baseMinForRewards

baseMinForRewards is defined in [line 86 in Comet.sol](https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol#L86)

![baseMinForRewards variable](https://static.wixstatic.com/media/935a00_5077de1f3c8c4d1bbf088b494916ef1f~mv2.png/v1/fill/w_666,h_63,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_5077de1f3c8c4d1bbf088b494916ef1f~mv2.png)

Compound doesn’t issue rewards to lenders or borrowers if there are less than 1 million dollars (1e12 USDC as USDC has 6 decimals) lent out. Similarly, a borrowed USDC will not accumulate COMP rewards if there is less than 1 million dollars borrowed.

![baseMinForRewards query in Etherscan](https://static.wixstatic.com/media/935a00_939dcaecbb994bce91587367eb0b9d56~mv2.png/v1/fill/w_315,h_94,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_939dcaecbb994bce91587367eb0b9d56~mv2.png)

### baseMinRewards exists to prevent accumulator overflow

Recall that the accumulated reward per token is inversely proportional to the amount of tokens staked. If the total supply of tokens staked is small, then the accumulator will accumulate rapidly, possibly overflowing too quickly.

If you are an auditor, this may be a fairly overlooked medium vulnerability because tests do not easily catch accumulator overflows. You need to make sure that the accumulator will not overflow for several years and this means either the reward rate needs to be small or the staked amount needs to be large.

## accrueInternal() revisited

The `trackingSupplyIndex` and `trackingBorrowIndex` are updated whenever `accrueInternal()` is called.

The code below implements the logic described in the above sections. The `if` conditions in <span style="color:Red">red boxes</span> prevent `trackingSupplyIndex` or `trackingBorrowIndex` from accumulating more rewards if the supply or borrow amount is below `baseMinForRewards`. The `baseTrackingSupplySpeed` and `baseTrackingBorrowSpeed` (blue boxes) are immutable variables, so the amount the indexes are incremented by only depends on `timeElapsed` and (inversely) to `totalSupplyBase` (or `totalBorrowBase`).

![accrueInternal](https://static.wixstatic.com/media/935a00_20d5aa07bc4e40dfb37ac76ab8b0d071~mv2.png/v1/fill/w_666,h_242,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_20d5aa07bc4e40dfb37ac76ab8b0d071~mv2.png)

You can think of the `baseTrackingSupplySpeed` and `baseTrackingBorrowSpeed` as the “reward per unit time.” When multiplied by `timeElapsed`, that computes the amount of rewards accumulated for a single participating USDC. Finally, dividing that result by `totalSupplyBase` or `totalBorrowBase` dilutes that USDC based on the total amount.

## baseSupplyTrackingSpeed and baseTrackingBorrowSpeed

These variables are analogous to the `rewardPerBlock` of MasterChef. They specify how quickly the accumulators described in the section above increase.

We can retrieve their values from the [Comet Etherscan Proxy contract](https://etherscan.io/address/0xc3d688B66703497DAA19211EEdff47f25384cdc3#readProxyContract).

![baseTrackingBorrowSpeed and baseTrackingSupplySpeed in Etherscan](https://static.wixstatic.com/media/935a00_815e13de31884a94948478651182a591~mv2.png/v1/fill/w_315,h_181,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_815e13de31884a94948478651182a591~mv2.png)

Both of these variables use trackingIndexScale for their decimals, and [per Etherscan, trackingIndexScale is 1e15](https://etherscan.io/address/0xc3d688B66703497DAA19211EEdff47f25384cdc3#readProxyContract):

![trackingIndexScale](https://static.wixstatic.com/media/935a00_97f90b95459d4710b35dd8fc9cddc283~mv2.png/v1/fill/w_315,h_194,al_c,lg_1,q_85,enc_auto/935a00_97f90b95459d4710b35dd8fc9cddc283~mv2.png)

Since they are 15 decimal [fixed point numbers](https://www.rareskills.io/post/solidity-fixed-point), their values are as follows:

baseTrackingSupplySpeed = 2.979166666666e-03

baseTrackingBorrowSpeed = 4.414467592592e-03

The variable definitions (with comments about the scale) from Comet.sol are screenshotted below

![tracking variable definitions](https://static.wixstatic.com/media/935a00_47d0cfc4e2b74c6bbe6481fd4076cb94~mv2.png/v1/fill/w_666,h_215,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_47d0cfc4e2b74c6bbe6481fd4076cb94~mv2.png)

## Tracking user-level rewards: baseTrackingAccrued and baseTrackingIndex

Like MasterChef, Compound Rewards accumulates rewards to an account when that account does a state-changing transaction. And also like MasterChef, the rewards the user accumulates are proportional to their balance and how much the “index” or “accumulator” changed since the last time the user did a state changing operation.

Let’s look at the user struct again

![UserBasic struct](https://static.wixstatic.com/media/935a00_86fa5ba62ff749fe8a2332512553f5a2~mv2.png/v1/fill/w_315,h_244,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_86fa5ba62ff749fe8a2332512553f5a2~mv2.png)

`baseTrackingIndex` is the value of `trackingSupplyIndex` or `trackingBorrowIndex` at the time the UserBasic storage struct was last updated, depending on if the account is a lender or borrower respectively. The delta between the current `trackingSupplyIndex` (or `trackingBorrowIndex`) and the user’s stored value of `baseTrackingIndex` determines how many rewards they will accumulate for that transaction. Consider the plot below

![trackingSupplyIndex and baseTrackingIndex for user](https://static.wixstatic.com/media/935a00_50083d76e6614ad3b007123c2d144551~mv2.jpg/v1/fill/w_666,h_499,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_50083d76e6614ad3b007123c2d144551~mv2.jpg)

Whenever a user does something that will change their principal, a call to the internal function `updateBasePrincipal()` is made. The function will determine how much the `trackingSupplyIndex` or `trackingBorrowIndex` has changed since the last update and accumulate rewards to the user’s `baseTrackingAccrued` accordingly. The function is shown below

![updateBasePrincipal() function](https://static.wixstatic.com/media/935a00_72001b68e881448d97c47b5e9dd96dc0~mv2.png/v1/fill/w_666,h_324,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_72001b68e881448d97c47b5e9dd96dc0~mv2.png)

In summary `baseTrackingIndex` is the value of the index when the user last updated. `baseTrackingAccrued` is the total rewards owed to the user since they participated in the protocol regardless of past claims which are negated with reward debt tracked in the reward contract.

### What is accrualDescaleFactor?

In the code above, we see the user’s accumulated rewards are divided by `accrualDescaleFactor`.

This allows both ETH and USDC to be tracked on the same scale. Since ETH has 18 decimals and USDC has 6 decimals, ETH baseTrackingAccrued is divided by 1e12 so that it effectively has the same number of “decimals” as USDC. This allows `baseTrackingAccrued` to track both assets on the same scale.

## Claiming rewards

To claim rewards, a user simply calls the `claim()` function in CometReward.sol. The `rewardsClaimed` mapping (<span style="color:Red">red box</span>) behaves like the rewardDebt from MasterChef.

![claim rewards function](https://static.wixstatic.com/media/935a00_e4cf5e213d7344999fcfdb0e0d167d99~mv2.png/v1/fill/w_666,h_576,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_e4cf5e213d7344999fcfdb0e0d167d99~mv2.png)

### What is the shouldAccrue argument for?

If someone is claiming rewards as the only action in a transaction, then `shouldAccrue` (<span style="color:Green">green box</span>) should be `true`. However, if it is after other function calls, then other state-changing function calls will call `accrueAccount()` making the another call unnecessary.

### getRewardAccrued() (CometRewards.sol)

In the <span style="color:#008aff">blue box</span> above, `getRewardAccrued` determining how much to pay the user. This simply queries the `baseTrackingAccrued` from the user struct in Comet. CometRewards then subtract it by their reward debt (`rewardsClaimed`) and pay the user the difference.

![getRewardsAccrued function](https://static.wixstatic.com/media/935a00_adebd04e15024b37980b464458fd0daa~mv2.png/v1/fill/w_666,h_183,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_adebd04e15024b37980b464458fd0daa~mv2.png)

## Quirks in the COMP token itself

The [COMP token](https://etherscan.io/token/0xc00e94Cb662C3520282E6f5717214004A7f26888#code) that the rewards contract distributes does not store balances as a uint256 like most ERC 20 tokens do, but rather as a uint96.

![COMP token balances mapping](https://static.wixstatic.com/media/935a00_a3f280ce42674fb8972c136125f81c03~mv2.png/v1/fill/w_666,h_40,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_a3f280ce42674fb8972c136125f81c03~mv2.png)

If you try to `transfer` or `approve` an amount greater than the uint96 maximum value, the transaction will revert.

![COMP transfer function reverts when amount is greater than 96 bits](https://static.wixstatic.com/media/935a00_ea5408c125d84282a803cbe03494f3d5~mv2.png/v1/fill/w_666,h_74,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_ea5408c125d84282a803cbe03494f3d5~mv2.png)

## Learn More With RareSkills

Please see our [Solidity bootcamp](https://www.rareskills.io/solidity-bootcamp) to learn more advanced smart contract development.

*Originally Published January 10, 2024*
