# ERC20 Snapshot

ERC20 Snapshot solves the problem of double voting.

If votes are weighed by the number of tokens someone holds, then a malicious actor can use their tokens to vote, then transfer the tokens to another address, vote with that, and so forth. If each address is a smart contract, then the hacker can accomplish all these votes in one transaction. A related attack is to use a flashloan to obtain a bunch of governance tokens, vote, then return the flash loan.

A similar problem exists for claiming airdrops. One could use their [ERC20](https://www.rareskills.io/post/erc20-votes-erc5805-and-erc6372) tokens to claim an airdrop, then transfer their tokens to another address, then claim the airdrop again.

Fundamentally, ERC20 snapshot provides a mechanism to defend against users transferring tokens and re-using token utility in the same transaction.

At first, snapshotting might seem like an intractable problem. The brute force, or naive, solution to this is to iterate through every address in the "balances" mapping of ERC20, then copy these to another mapping. It isn't possible to natively iterate through a mapping in ERC20, so the coder would have to use an [enumerable map](https://docs.openzeppelin.com/contracts/3.x/api/utils#EnumerableMap) — a map with an array that keeps track of all the keys.

As one can imagine, this O(n) operation would be extremely gas intensive.

There is a saying in computer science that "Every problem in computer science can be solved with another level of indirection" and that's how ERC20 snapshots solves it.

## Efficient but Naive Solution

Let's take the example of the balances mapping.
Here is a buggy [solidity](https://www.rareskills.io/solidity-bootcamp) solution, but a step in the right direction.

```solidity
balances[snapshotNumber][user]
```

In this case, `snapshotNumber` is a counter that starts at zero and increments by one every time a snapshot is done.

Going back to our voting example, we create a snapshot at a particular point in time, let everyone go about their business, then create another snapshot. At the time of voting, we use the **previous** snapshot since the current snapshot can still be changed by transferring tokens.

This way, we can query someone's balance by supplying both the `snapshotNumber` and their address at the snapshot we care about. Since we know the current snapshot, `balanceOf` is simply the balances at the most recent snapshot.

Ah, but there is a problem! Every time we do a snapshot, everyone's balances are set to zero! It is possible to solve this with some accounting — just track the last snapshot the user transacted at, but this quickly gets complicated as the engineer tries to cover all the corner cases.

## OpenZeppelin solution

This is how OpenZeppelin accomplishes it. [code](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/release-v4.9/contracts/token/ERC20/extensions/ERC20Snapshot.sol
)

Each balance stores a struct

```solidity
struct Snapshots {
    uint256[] ids;
    uint256[] values;
}

mapping(address => Snapshots) private _accountBalanceSnapshots;

function balanceOfAt(address account, uint256 snapshotId) public view virtual returns (uint256) {
    (bool snapshotted, uint256 value) = _valueAt(snapshotId, _accountBalanceSnapshots[account]);

    return snapshotted ? value : balanceOf(account);
}
```

Inside the user balances, we store a struct which has an array of ids and values. The array of ids is a monotonically increasing snapshot id, and the values are the balance when that id was the active snapshot.

### Taking a snapshot

Here is the snapshot function. It simply increments the current snapshot id.

```solidity
function _snapshot() internal virtual returns (uint256) {
    _currentSnapshotId.increment();

    uint256 currentId = _getCurrentSnapshotId();
    emit Snapshot(currentId);
    return currentId;
}
```

When a user makes a transfer in the new snapshot, the `_beforeTokenTransfer` hook is called, which has the following code.

Both the receiver and the sender have `_updateAccountSnapshot` called.

```solidity
// Update balance and/or total supply snapshots before the values are modified. This is implemented
// in the _beforeTokenTransfer hook, which is executed for _mint, _burn, and _transfer operations.
function _beforeTokenTransfer(address from, address to, uint256 amount) internal virtual override {
    super._beforeTokenTransfer(from, to, amount);

    if (from == address(0)) {
        // mint
        _updateAccountSnapshot(to);
        _updateTotalSupplySnapshot();
    } else if (to == address(0)) {
        // burn
        _updateAccountSnapshot(from);
        _updateTotalSupplySnapshot();
    } else {
        // transfer
        _updateAccountSnapshot(from);
        _updateAccountSnapshot(to);
    }
}
```

This is the `_updateAccountSnapshot` function that gets called

```solidity
function _updateAccountSnapshot(address account) private {
    _updateSnapshot(_accountBalanceSnapshots[account], balanceOf(account));
}
```

Which in turn calls `_updateSnapshot` . The definition is below

```solidity
function _updateSnapshot(Snapshots storage snapshots, uint256 currentValue) private {
    uint256 currentId = _getCurrentSnapshotId();
    if (_lastSnapshotId(snapshots.ids) < currentId) {
        snapshots.ids.push(currentId);
        snapshots.values.push(currentValue);
    }
}
```

Because the `currentId` was just incremented, the if statement will be true. Inside the snapshots array, the current balance will be appended. Because this was called in the `_beforeTokenTransfer` hook, this reflects the balance before it gets changed.

Hence, once the snapshot ID increases, any transfers that happen after the snapshot transaction will store the balances *before* the transaction takes place and store that into the array. This effectively "freezes" everyone's current balance because any transfer that happens after the snapshot causes the "old" value to get stored. What happens if two snapshots happen, but an address does not transact during those snapshots? In that case, the snapshot ids will not be contiguous.

Because of this, we cannot access an account's balance at a snapshot by doing "ids[snapshotId]". Instead, [binary search](https://en.wikipedia.org/wiki/Binary_search_algorithm) is used to find the snapshot id the user is requesting. If the id is not found, then we use the previous adjacent snapshot value. For example, if we want to know a user's balance at snapshot 5, but they didn't transfer tokens during snapshots 3 and 4, we would look at snapshot 2.

## Total supply is tracked the same way

The reader may note that the struct Snapshots has seemingly overgeneric variable names, like ids and values. Shouldn't it just be named "balance" to be more precise?
ERC20 Snapshot keeps track of the total supply using the same strategy, so the variable names capture the fact that the same struct is used both for tracking user balances and the total supply.

Only mint and burn change the total supply, so when these functions are invoked, the struct storing the total supply is checked to see if the snapshot has changed before updating these values.

Note that historic allowance values are not snapshotted.

## Added gas cost

Regular transfers are more expensive because we check if the last id in the ids of the user matches the current snapshot and add a new id if that is not the case. Appending to the array of ids and values will incur two extra SSTOREs. When a new snapshot occurs, the first transfer to or from an address will be more expensive. But the second transaction will cost approximately the same as a transfer in a regular ERC20 token.

## Getting hacked

If someone takes out a flash loan and creates a snapshot in the same transaction, they can artificially inflate their voting power. If the tokens can be borrowed at a low interest rate, and an attacker knows when the next snapshot will occur, they can borrow tokens leading up to the snapshot to accomplish something similar. However, a flash loan will not be a viable way to inflate voting power, since they will need the balance to stay high during a *separate* snapshot transaction.

## Tallying votes

This is simply the balance of an address divided by the total supply, all at a particular snapshot.

*Originally Published Feb 22, 2023*
