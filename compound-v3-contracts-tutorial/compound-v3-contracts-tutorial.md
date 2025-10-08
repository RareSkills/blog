# The Architecture of the Compound V3 Smart Contract

## Introduction and prerequisites

Compound is one of the most significant lending protocols in DeFi, having inspired the design of nearly every lending protocol on several blockchains. This article explains the smart contract architecture of V3.

Since Compound is a lending protocol, we assume the reader is familiar with [how interest rates work in DeFi](https://www.rareskills.io/post/aave-interest-rate-model) and are familiar with [liquidations and collateral](https://www.rareskills.io/post/defi-liquidations-collateral) in the context of DeFi loans.

Unlike Compound V2, each instance of Compound only lends one asset. That is, you can only borrow USDC from the USDC market and can only borrow ETH from the ETH market -- you cannot borrow anything else. To borrow these assets, you need to supply collateral according to the collateralization ratio specified by the market (and that ratio is set by governance).

The borrowable asset is called the **base asset** in Compound V3. Because USDC is the most popular asset to borrow and lend, we will use USDC as the base asset in our running examples. The smart contract that handles the core logic of lending and borrowing is referred to as **Comet** in their literature and codebase. Compound has instances on Ethereum mainnet and on various L2s: Polygon, Base, and Arbitrum at this time of writing. The list of available [Compound V3 markets is here](https://app.compound.finance/markets).

This article gives a high level overview of how to use this smart contract and the architecture of the Solidity files.

## Part 1: Using Compound

There are three major actions that can be taken with Compound V3:

1.  lending the base asset

2.  supplying collateral and borrowing the base asset

3.  liquidating under-collateralized loans.

### Lending USDC

The following video shows the actions for lending USDC on the Polygon network ([USDC on Polygon market link](https://app.compound.finance/?market=usdc.e-polygon)).

<video src="https://video.wixstatic.com/video/935a00_8728e7e7504a44be895a26c450693ac9/480p/mp4/file.mp4" style="width: 100%; height: 100%;" autoplay loop muted controls></video>

People who participate in the protocol are awarded COMP token rewards. The account above has accrued 0.0004 COMP rewards so far as indicated by the <span style="color:Pink">pink arrow</span>. The rewards have not been claimed yet. Additionally, it has accrued over 6 cents in interest so far (4,602.0649 - 4,602.00 = 0.0649).

![An image of Lending USDC dashboard](https://static.wixstatic.com/media/935a00_5066cbdd8a0a4a7b820794af8655bf22~mv2.png/v1/fill/w_666,h_397,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_5066cbdd8a0a4a7b820794af8655bf22~mv2.png)

The mismatch between the prices in the <span style="color:Green">green box</span> and the <span style="color:#c1c146">yellow box</span> is because USDC is not always exactly $1.00. The chart below shows Chainlink's recent [oracle prices of USDC / USD on Polygon](https://data.chain.link/polygon/mainnet/stablecoins/usdc-usd) and the reader can see it is not always exactly one dollar.

![An image of USDC/USD charts](https://static.wixstatic.com/media/935a00_f3c07aeb24b642059d33b2b868af9504~mv2.png/v1/fill/w_666,h_293,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_f3c07aeb24b642059d33b2b868af9504~mv2.png)

### Borrowing USDC

The following video shows the procedure for borrowing. This time the account supplied 0.06 ETH ($110) as collateral and borrowed $100 of USDC. This was done on the [BASE L2 lending market](https://app.compound.finance/?market=usdbc-basemainnet).

<video src="https://video.wixstatic.com/video/935a00_ac5f26fe0dae4bd0b666dbb0ea005153/480p/mp4/file.mp4" style="width: 100%; height: 100%;" autoplay loop muted controls></video>

Checking the browser wallet, the account now has 100 Base USDC in it. USDCbC is USDC bridged to Base.

![An image of the browser wallet](https://static.wixstatic.com/media/935a00_99c5ffa38b6d40a2b3cf451f1466b732~mv2.png/v1/fill/w_315,h_425,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_99c5ffa38b6d40a2b3cf451f1466b732~mv2.png)

The following video shows the user paying back the USDC (before interest significantly accumulated) and withdrawing the ETH collateral.

## Understanding net borrow and net lending rates

![An image of net borrow and net lending rates](https://static.wixstatic.com/media/935a00_83b3da78c73f42fb92d565a6a8779e5e~mv2.png/v1/fill/w_666,h_395,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_83b3da78c73f42fb92d565a6a8779e5e~mv2.png)

In the screenshot above, note that the Net Supply APR (lending interest rate) is higher than the Net Borrow APR (<span style="color:#c1c146">yellow circles</span>). This normally isn’t possible because lenders cannot earn more than what borrowers are paying. Compound is including the value of COMP rewards into the calculation. Specifically, the borrowers currently pay 6.71% interest (<span style="color:Red">top red circle</span>) but earn 2.58% interest in COMP (<span style="color:#008aff">top blue circle</span>). 6.71 - 2.58 = 4.13%, which is the Net Borrow APR.

Similarly, lenders currently earn 6.47% from lending USDC (<span style="color:Red">lower red circle</span>), but also earn an additional 4.63% from the value of COMP rewards (<span style="color:#008aff">lower blue circle</span>). The sum of those two is 11.10%.

Since borrowers are paying 6.71% interest in USDC and lenders are collecting 6.47% interest, 0.24% interest in USDC is going to the protocol.

COMP rewards rates change subject to governance votes.

## Part 2: Navigating the codebase

Compound V3 has 4,304 lines of Solidity code, not counting comments or blank spaces.

![An image for Navigating the codebase](https://static.wixstatic.com/media/935a00_41ec54f529c340b486d30fed38fff067~mv2.png/v1/fill/w_315,h_159,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_41ec54f529c340b486d30fed38fff067~mv2.png)

## Compound V3 architecture from 10,000 feet

Below we have screenshotted the [GitHub repo of Compound V3](https://github.com/compound-finance/comet/tree/main/contracts).

-   All of the files highlighted in <span style="color:Green">green</span> hold the core borrowing and lending functionality. The inheritance relationship between them will be shown later. **Comet is the primary lending and borrowing smart contract of Compound V3.**
-   All the files highlighted in <span style="color:#008aff">blue</span> form the smart contract that deploys new Comet instances during an upgrade. Again, the inheritance relationship between them will be shown later.
-   The <span style="color:Pink">pink</span> highlighted file is the contract that distributes rewards

![An image of GitHub repo Compound V3](https://static.wixstatic.com/media/935a00_496ce460835a4cffb299bb4b4b0ee060~mv2.jpg/v1/fill/w_666,h_364,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_496ce460835a4cffb299bb4b4b0ee060~mv2.jpg)

The following diagram summarizes the deployed smart contracts that make up Compound V3. This is only a high level overview, a more detailed one will be given later. Note that the color coding matches the highlights above. Specifically, the primary lending and borrowing contract (Comet) is <span style="color:Green">green</span>, the contracts related to deploying new Comet instances are blue, and the reward contract is <span style="color:Pink">pink</span>.

![An image of a high level overview Compound V3](https://static.wixstatic.com/media/935a00_5e3774507c414cdcb03d6e7a01c6ac60~mv2.jpeg/v1/fill/w_666,h_499,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_5e3774507c414cdcb03d6e7a01c6ac60~mv2.jpeg)

Most users will interact with Compound V3 through the comet proxy (not shown in GitHub) or with the <span style="color:Pink">rewards</span> contract to earn COMP for participating as a lender or borrower. All of the user-facing logic is in the <span style="color:Green">comet</span> contract where the comet proxy delegates its functionality to.

The <span style="color:#008aff">configuration and factory</span> contracts are to deploy new comet instances when governance votes for an upgrade.

The smart contracts with an asterisk (*) have several ancestor contracts. In the next section, we show the inheritance chain for Comet.

## Compound V3 has an unusual method of updating parameters based on governance

Interest rate models can be changed by governance if crypto-economic conditions change, and so can the liquidation factors. **Compound stores all of this information in immutable variables, not storage. To alter these values, a new Comet instance must be deployed, and the proxy is updated to the new implementation contract.**

This may seem like an odd design choice, but it has several advantages:

-   immutable variables are much more gas efficient than storage variables
-   The core contracts don’t need to clutter themselves with setter functions

We will examine the lifecycle of a parameter change later.

## Comet Inheritance

Comet’s ancestors are shown in the diagram below. We will give a high-level overview of each of the ancestor contracts in this section.

![An image of Comet Inheritance](https://static.wixstatic.com/media/935a00_e9a7d9296860483389f5685d36b6af94~mv2.jpg/v1/fill/w_666,h_499,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_e9a7d9296860483389f5685d36b6af94~mv2.jpg)

### CometMath.sol (top left <span style="color:Gray">gray ellipse</span>)

Comet math simply contains a bunch of functions for casting unsigned integers to unsigned integers of lower bit value and reverting if it would cause an overflow. For example, if we cast from [uint256](https://www.rareskills.io/post/uint-max-value-solidity) to uint104, but the value of the uint256 is larger than what can be stored in uint104, it will revert. You’ll see functions like `safe64` scattered throughout the codebase. Hopefully these are easy enough to understand without further explanation. The file is small, so we show it in its entirety here:

![An image of CometMath.sol](https://static.wixstatic.com/media/935a00_77dd92cf4e554066a6eff8dc2fab727d~mv2.png/v1/fill/w_315,h_675,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_77dd92cf4e554066a6eff8dc2fab727d~mv2.png)

### CometStorage.sol (<span style="color:Red">red ellipse</span>)

Every single storage variable used by Comet is defined in [CometStorage.sol](https://github.com/compound-finance/comet/blob/main/contracts/CometStorage.sol) and nowhere else. That is, no other contract in the Comet inheritance chain besides CometStorage has any storage variables.

### CometCore.sol (second row <span style="color:Gray">gray ellipse</span>)

[CometCore.sol](https://github.com/compound-finance/comet/blob/main/contracts/CometCore.sol) defines how Compound tracks and interprets the accumulation of interest for lenders and borrowers. It also defines some global constants. We will dive into this file more when we discuss Principal Value and Present Value.

### CometMainInterface.sol (left <span style="color:#008aff">blue ellipse</span>)

As the name implies, [CometMainInterface.sol](https://github.com/compound-finance/comet/blob/main/contracts/CometMainInterface.sol) is simply an interface file, except with the quirk that it is defined as an abstract contract. Since interfaces are self-explanatory, we omit discussion of this.

### Comet.sol (<span style="color:Green">green ellipse</span>)

[Comet.sol](https://github.com/compound-finance/comet/blob/main/contracts/Comet.sol) is the star of the show. It provides all the public facing functions for users to lend, borrow, repay, and liquidate loans.

### CometExt is an extension to Comet via [delegatecall](https://www.rareskills.io/post/delegatecall)

To avoid hitting the 24 KB deployment limit, Comet offloads several extra functions to CometExt using the [fallback-extension pattern](https://www.rareskills.io/post/fallback-extension-pattern). For example, the function `name()` is not in Comet.sol and thus [cannot be seen on Etherscan](https://etherscan.io/address/0xc3d688B66703497DAA19211EEdff47f25384cdc3#readProxyContract).

However, if we call that function via [Foundry](https://www.rareskills.io/post/foundry-testing-solidity) cast, we can see the contract behaves as if it exists.

![An image showing CometExt as an extension to Comet via delegatecall](https://static.wixstatic.com/media/935a00_fc0907ea4298469791e4860a442f6168~mv2.png/v1/fill/w_666,h_57,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_fc0907ea4298469791e4860a442f6168~mv2.png)

What happens is that function call hits the fallback function and delegates the call to CometExt, which has a function called `name()` in it.

It is important that CometExt respects the storage layout of Comet and **extends it, without clashing.** This is achieved by having CometExt mimic Comet’s inheritance structure.

Remember, _inheritance is just a semantic description — when contracts are deployed, the entire inheritance chain becomes one contract. It is not possible for a deployed contract to inherit from another deployed contract._

A more verbose diagram showing the relationship between Comet and CometExt is shown here

![A verbose diagram showing the relationship between Comet and CometExt](https://static.wixstatic.com/media/935a00_9b051544b12346bcb0e6cb16879e59ef~mv2.jpg/v1/fill/w_666,h_499,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_9b051544b12346bcb0e6cb16879e59ef~mv2.jpg)

## Rewards Issuance

There is only one contract for handling non-interest reward issuance show in the <span style="color:Pink">pink box</span>.

![An image for Rewards Issuance](https://static.wixstatic.com/media/935a00_e97351b8eb65489781fee25ecb182341~mv2.png/v1/fill/w_666,h_329,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_e97351b8eb65489781fee25ecb182341~mv2.png)

Lenders and borrowers are rewarded with COMP tokens for their participation in the ecosystem. The Comet contract keeps track of participation, but it does not handle reward issuance. The Reward contract reads the state of Comet and issues COMP tokens. The rate at which rewards are issued are parameters inside the Reward contract that are set by governance.

## Lifecycle of a parameter update

As mentioned earlier, Comet is parameterized by immutable variables. Changing these parameters requires updating the proxy to point to a new implementation deployed by the Configuration contracts.

In practice, changing the interest rate curves is very infrequent. The mainnet contract has only undergone three interest rate curve updates:

[https://compound.finance/governance/proposals/162](https://compound.finance/governance/proposals/162) (May 30, 2023)

[https://compound.finance/governance/proposals/168](https://compound.finance/governance/proposals/168) (July 17, 2023)

[https://compound.finance/governance/proposals/201](https://compound.finance/governance/proposals/201) (Dec 11, 2023)

Interest rates are not the only parameters that can change -- other notable ones include changing liquidation ratios and even permitted collateral assets. The curious reader can peruse the governance proposals.

The GIF below shows how the Configuration smart contract is structured and how governance deploys new Comet instances with updated parameters.

![An image of Configuration smart contract structure](https://static.wixstatic.com/media/935a00_699ec73b9eb147898b1f6a45c7483232~mv2.gif)

## CometConfiguration.sol and ConfiguratorStorage.sol

CometConfiguration.sol defines the struct which parameterize the entire behavior of Comet. CometStorage simply stores those structs.

If you’ve read our article on How DeFi Interest Rates Work and DeFi Collateral and Liquidations, then hopefully the variable names in the struct are mostly self-explanatory. ConfiguratorStorage inherits these definitions and stores the structs in mappings.

**These storage variables are not part of Comet. The Configurator contract deploys Comet instances using these stored configurations.**

Below we show a snapshot of the code for [CometConfiguration](https://github.com/compound-finance/comet/blob/main/contracts/CometConfiguration.sol) and [ConfiguratorStorage](https://github.com/compound-finance/comet/blob/main/contracts/ConfiguratorStorage.sol).

![A snapshot of the code for CometConfiguration and ConfiguratorStorage](https://static.wixstatic.com/media/935a00_f46e1a66450e4277893c4b9e38a6d5ae~mv2.jpeg/v1/fill/w_666,h_369,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_f46e1a66450e4277893c4b9e38a6d5ae~mv2.jpeg)

This is a much more preferable way to deploying new Comet instances than supplying an extremely large struct in the calldata.

When a new Comet instance is deployed with an update, we only need to update a particular storage variable, leaving the others unchanged. For example, if we change the liquidation threshold for the collateral wBTC, only that storage variable in the configurator will be affected.

## Configurator.sol

The Configurator.sol contract inherits ConfigurationStorage and provides governance-only setter functions. The [Configurator.sol file](https://github.com/compound-finance/comet/blob/main/contracts/Configurator.sol) is quite large, so we don’t show the whole thing. However, just by looking at the events defined in it, we can get a good idea of what the contract does — update individual fields in the struct that parameterize the deployment of a new Comet instance.

![An image of Configurator.sol showing what the contract does](https://static.wixstatic.com/media/935a00_59f421ba5e414bfcb641cb2f43c1e028~mv2.png/v1/fill/w_666,h_576,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_59f421ba5e414bfcb641cb2f43c1e028~mv2.png)

Configurator.sol also holds the `deploy()` function to deploy new Comet instances.

![An image of Configurator.sol showing deploy()](https://static.wixstatic.com/media/935a00_72fea30104b94d57a6733bea379ce4c8~mv2.png/v1/fill/w_666,h_198,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_72fea30104b94d57a6733bea379ce4c8~mv2.png)

`CometFactory` in the code above is defined in [CometFactory.sol](https://github.com/compound-finance/comet/blob/main/contracts/CometFactory.sol) and is quite small, so we show the file in its entirety. The function "clone" is a bit misleading because it does not create [proxy clones](https://www.rareskills.io/post/eip-1167-minimal-proxy-standard-with-initialization-clone-pattern) -- it creates a new instance.

![An image showing clone function](https://static.wixstatic.com/media/935a00_dd23c1f2937341c9a09d224172c84562~mv2.png/v1/fill/w_666,h_290,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_dd23c1f2937341c9a09d224172c84562~mv2.png)

We refer to the same animation from earlier to tie all of our previous discussion together.

![A GIF tying all previous discussion together](https://static.wixstatic.com/media/935a00_699ec73b9eb147898b1f6a45c7483232~mv2.gif)

## Real Example of a Parameter Change

Let’s use governance [Proposal 162](https://compound.finance/governance/proposals/162) as an example. We see it called some setter functions on the Configurator to update the parameters, then deployed a new Comet instance in the final step (step 8).

![An image of Real Example of a Parameter Change](https://static.wixstatic.com/media/935a00_576994ce44684fcdabc27cf08ab29c09~mv2.png/v1/fill/w_666,h_424,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_576994ce44684fcdabc27cf08ab29c09~mv2.png)

## All Together

Below we show the relationship between all the files and how they interact with each other. The interface files are ignored.

![An image showing the relationship between all the files and how they interact with each other](https://static.wixstatic.com/media/935a00_d493488bd1db42a29111edf14f7fbcf6~mv2.jpg/v1/fill/w_666,h_206,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_d493488bd1db42a29111edf14f7fbcf6~mv2.jpg)

## Learn more with RareSkills

Please see our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) to learn more.

*Originally Published January 3, 2024*
