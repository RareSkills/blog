# ERC20 Votes: ERC5805 and ERC6372

![ERC20 Votes](https://static.wixstatic.com/media/935a00_57a870a4688e424684737ab551a6bf88~mv2.webp/v1/fill/w_666,h_662,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_57a870a4688e424684737ab551a6bf88~mv2.webp)

ERC20 Votes

Knowledge of ERC20 Snapshot is assumed, please refer to our article on [ERC20 Snapshot](https://www.rareskills.io/post/erc20-snapshot) for an introduction to the subject.

ERC20 Votes does not actually handle conducting the poll, it’s still a regular ERC20 token with snapshot and delegated voting abilities. Voting is usually handled by governance contracts.

Delegation means an address can loan their voting power to another address without transferring their tokens to that address.

Unlike [ERC20 Snapshot](https://www.rareskills.io/post/erc20-snapshot), it does not keep snapshots of the token balances, but keeps snapshots of the voting power of the addresses.

There are a couple motivations for this:

-   This allows passive shareholders to participate in governance by delegating their voting power
-   Accounts with small balances can vote without spending gas, because the delegatee is using voting on their behalf. This saves gas at an ecosystem level. If all participants have delegated to 5 delegatees, then only 5 votes happen for an item to be voted on, instead of possibly thousands.
-   And of course, snapshotting (checkpointing) is necessary to prevent double voting, just like ERC20 Snapshot does.

ERC20 Votes inherits from ERC20, ERC6372, and ERC5805. This means it has all the functionality of an ERC20 token with additional attributes described below.

## ERC5805 functionality

### `getVotes(address delegate)`

This function takes the address of an account and returns the total voting power it has. This can be greater than balanceOf(delegate) if other addresses have delegated their voting power to it.

### `delegate(address delegatee)`

This function allows `msg.sender` to delegate their voting power to delegatee and the delegatee will vote on behalf of `msg.sender`. Note that the tokens are not transferred to delegatee, only the voting power. Delegation can be changed or revoked at any time by calling delegate again with a different delegatee or with `address(0)`.

Delegation is all or nothing. There is no option to delegate a fraction of voting power.

It is important to note that an address must delegate to itself before its votes are counted. This quirk is introduced for gas efficiency reasons.

### `delegates(address account)`

This function returns the account that `account` in the argument has delegated its voting power to.

### `delegateBySig(address delegatee, uint256 nonce, uint256 expiry, uint8 v, bytes32 r, bytes32 s)`

The function delegateBySig allows for a user to delegate voting power through a gasless transaction and have another account pay the gas fee and execute the transaction.

The expiration sets the time over which the delegation is valid, and v, r, and s are the components of the Elliptic Curve Digital Signature.

This signature expected to be in the [EIP 712](https://eips.ethereum.org/EIPS/eip-712) format. Internally, the contract increments the nonce per address, which is described next.

### `nonces(address account)`

Signatures require a nonce to prevent replay attacks, so the internal tracker of the nonce is exposed through this function. This enables the signer to know which value they should sign next.

### `getPastVotes(account, timepoint)`

As discussed in our [ERC20 Snapshot article](https://www.rareskills.io/post/erc20-snapshot), unless account balances are snapshotted, a double voting attack can occur. This is where a ballot contract would look at a past snapshot.

Unlike ERC20 snapshot, a snapshot is not triggered by an address calling the snapshot function. A snapshot is triggered per account when one of these events happens:

-   minting
-   burning
-   transfer
-   someone delegates their votes

Each time one of these events happens, a struct containing the voting power and the timestamp is appended to an array that stores that user’s voting power history.

Like ERC20 Snapshot, ERC20 Votes will do a binary search over the timestamps to find the earliest checkpoint after the timepoint. The voting power at that timepoint is returned.

This means, unlike ERC20 snapshot, there is no "global" snapshot id. If you want to query a point in the past, you select a timestamp or block number and supply that as the "timepoint" to the function.

### **Events**

ERC5805 has two events which signify what their name sounds like: `DelegateChange` and `DelegateVotesChanged`.

### **ERC5805 is an interface, not a token**

In this article, we are explaining ERC20 Votes, but that doesn't mean ERC5805 has to be a fungible token. It could be an NFT, or even an accounting of votes managed some other way, such as a centralized entity assigning votes to addresses, but wanting to keep an immutable history of how voting power was distributed.

## **ERC6372**

We somewhat assumed that the contract was using `block.timestamp` for bookkeeping on how to record the checkpoint. However, some contracts might use the `block.number` or some monotonically increasing function of these global variables.

ERC6372 is a standard to allow contracts to query what kind of "clock" the contract is using. It has two functions:

### `clock()`

This returns a uint48 which could be the block number or block timestamp or a function of these. uint48 was chosen because it has enough bits to represent all sensible representations of time and the block number farther into the future than humanity has recorded history.

### `CLOCK_MODE()`

Yes, a snake-case all upper case function is very unusual in Solidity, but that is what the EIP specifies. This returns a string that tells the reader what unit the clock uses.

-   If it is a timestamp, it will be
"mode=timestamp".
-   If it is a block number it will be `mode=blocknumber&from=default`.

This means it is just using `block.number` variable. If it was starting from another block, it must specify the chain id and the block number it is starting at. For example, Avalanche’s chain ID is 43114, so the response from `CLOCK_MODE()`` would be `mode=blocknumber&from=43114:100` if it was starting from the 100th block.

EIP 6372 has no events.

For more detail, see the [EIP](https://eips.ethereum.org/EIPS/eip-6372), which is actually quite readable. Note this EIP is not finalized.

## Summary of differences with ERC20 Snapshot

### Notion of time

-   ERC20 Votes has an explicit notion of time
-   ERC20 Snapshot uses incrementing ids that increase with time as a byproduct of being a counter

### What gets recorded

-   ERC20 Votes snapshots voting power
-   ERC20 Snapshot keeps track of balances

### When are the checkpoints updated

-   ERC20 Votes updates whenever a delegation or transfer happens
-   ERC20 Snapshot need an explicit call to “\_snapshot()”

## Whether to use ERC20 Snapshot or Votes

The choice to use ERC20 Snapshot or Voting depends mostly on whether there is a need to delegate votes, or more abstractly, some right the ERC20 token confers.

## Learn more

See our advanced [solidity programming bootcamp](https://www.rareskills.io/solidity-bootcamp) and our other [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) offerings.

*Originally Published February 23, 2023*
