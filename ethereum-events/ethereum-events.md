# Solidity Events

Solidity events are the closest thing to a `print` or `console.log` statement in Ethereum. We will explain how they work, best practices for events, and go into a lot of technical details often omitted in other resources.

Here is a minimal example to emit a solidity event.

```solidity
contract ExampleContract {

  // We will explain the significance of the indexed parameter later.
  event ExampleEvent(address indexed sender, uint256 someValue);

  function exampleFunction(uint256 someValue) public {
    emit ExampleEvent(sender, someValue);
  }
}
```

Perhaps the most well-known events are those emitted by ERC20 tokens when they are transferred. The sender, receiver, and amount are recorded in an event.

Isn’t this redundant? We can already look through the past transactions to see the transfers, and then we could look into the calldata to see the same information.

This is correct; one could delete events and have no effect on the business logic of the smart contract. However, this would not be an efficient way to look at history.

## Retrieving transactions faster

The [Ethereum](https://www.rareskills.io/ethereum-gas-price-calculator) client does not have an API for listing transactions by "type." Here are your options if you want to query historical transactions:

-   getTransaction
-   getTransactionFromBlock

`getTransactionFromBlock` can only tell you what transactions occurred on a particular block, it cannot target smart contracts across multiple blocks.

`getTransaction` can only inspect transactions you know the transaction hash for.
Events on the other hand can be retrieved much more easily. Here are the Ethereum client options:

-   `events`
-   `events.allEvents`
-   `getPastEvents`

Each of these require specifying the smart contract address the querier wishes to examine, and returns a subset (or all) of the events a smart contract emitted according to the query parameters specified.

To summarize: Ethereum does not provide a mechanism to get all transactions for a smart contract, but it does provide a mechanism for getting all events from a smart contract.

Why is this? Making events quickly retrievable requires additional storage overhead. If Ethereum did this for every transaction, this would make the chain considerably larger. With events, solidity programmers can be selective about what kind of information is worth paying the additional storage overhead for, to enable quick off-chain retrieval.

## Listening to Events

Events are intended to be consumed off-chain.

Here is an example of using the API described above. In this code, the client subscribes to events from a smart contract.

### Example 1: Listening to ERC20 Transfer events.

This code triggers a callback every time an [ERC20](https://www.rareskills.io/post/erc20-votes-erc5805-and-erc6372) token emits a transfer Event.

```javascript
const { ethers } = require("ethers");

// const provider = your provider

const abi = [
  "event Transfer(address indexed from, address indexed to, uint256 value)"
];

const tokenAddress = "0x...";
const contract = new ethers.Contract(tokenAddress, abi, provider);

contract.on("Transfer", (from, to, value, event) => {
  console.log(`Transfer event detected: from=${from}, to=${to}, value=${value}`);
});
```

### Example 2: Filtering an ERC20 approval for a specific address

If we want to look at events retroactively, we can use the following code. In this example, we look to the past for Approval transactions in an ERC20 token.

```javascript
const ethers = require('ethers');

const tokenAddress = '0x...';

const filterAddress = '0x...';

const tokenAbi = [
  // ...
];

const tokenContract = new ethers.Contract(tokenAddress, tokenAbi, provider);

// this line filters for Approvals for a particular address.
const filter = tokenContract.filters.Approval(filterAddress, null, null);

tokenContract.queryFilter(filter).then((events) => {
  console.log(events);
});
```

If you wanted to look for a trade between to particular addresses (if such a transaction exists), the ethers js javscript code would be as follows:

```javascript
tokenContract.filters.Transfer(address1, address2, null);
```

The `null` in the code above means "match any value for this field." For the transfer event, we are matching any amount.

Here is a similar example in [web3.js](https://www.rareskills.io/post/web3-js-tutorial). Note that the `fromBlock` and `toBlock` query parameters are added, and we will demonstrate the ability to listen to multiple addresses being the sender. The addresses are combined with the "OR" condition.

```javascript
const Web3 = require('web3');

const web3 = new Web3('https://rpc-endpoint');
const contractAddress = '0x...'; // The address of the ERC20 contract
const contractAbi = [
  {
    "anonymous": false,
    "inputs": [
      {
        "indexed": true,
        "name": "from",
        "type": "address"
      },
      {
        "indexed": true,
        "name": "to",
        "type": "address"
      },
      {
        "indexed": false,
        "name": "value",
        "type": "uint256"
      }
    ],
    "name": "Transfer",
    "type": "event"
  }
];
const contract = new web3.eth.Contract(contractAbi, contractAddress);

const senderAddressesToWatch = ['0x...', '0x...', '0x...']; // The addresses to watch for transfers from

const filter = {
  fromBlock: 0,
  toBlock: 'latest',
  topics: [
    web3.utils.sha3('Transfer(address,address,uint256)'),
    null,
    senderAddressesToWatch,
  ]
};

contract.getPastEvents('Transfer', {
  filter: filter,
  fromBlock: 0,
  toBlock: 'latest',
}, (error, events) => {
  if (!error) {
    console.log(events);
  }
});
```

Ranged queries are not possible. You cannot specify a filter that says "give me all transactions where the amount falls between this lower bound and that upper bound." You must get all the events then filter them in the client code.

### Example 3: Creating a leaderboard without storing the entries

Consider a smart contract with a donation function, and you want the frontend to rank donations by amount given. Here is an inefficient solution:

```solidity
contract Donations {
    struct Donation {
        address donator;
        uint256 amount;
    }
    Donation[] public donations; // frontend queries this
    
    fallback() external payable {
        donations.push(Donation({
            donator: msg.sender,
            amount: msg.value
        }));
    }
    
    // more functions for the owner to withdraw
}
```

If the donations don’t need to be read on-chain, this is the naïve solution, as it will significantly increase the gas cost for the person sending Ethereum to the contract.

This is a better solution using events:

```solidity
contract Donations {
    event Donation(address indexed donator; uint256 amount);
    fallback() external payable {
        emit Donation(msg.sender, msg.value);
    }
    // more functions for the owner to withdraw
}
```

The frontend can simply query the smart contract for all `Donation` events then sort them by the amount field.

Events are stored in the blockchain’s state, they are not ephemeral. Therefore, there is no need to worry about the client `missing` an event. They simply re-query events for the contract.

## Solidity indexed events vs non-indexed events

The example above works because the `Approve` (and `Transfer`) event in ERC20 sets the sender to be indexed. Here is the declaration in Solidity.

```
event Approval(address indexed owner, address indexed spender, uint256 value);
```

If the `owner` argument was not indexed, the javascript code earlier would silently fail. The implication here is that you cannot filter for ERC20 events that have a specific value for the transfer, because that is not indexed. You must pull in all the events and filter them javascript-side; it cannot be done in the Ethereum client.

An indexed argument for an event declaration is called a **topic**.

## Solidity events best practices

The generally accepted best practice for events is to log them whenever a consequential state change happens. Some examples include:

-   Changing the owner of the contract
-   Moving ether
-   Conducting a trade

Not every state change requires an event. The question the Solidity developers should ask themselves is “would someone have an interest in retrieving or discovering this transaction quickly?”

### Index the right event parameters

This will requires some subjective judgement. Remember, an unindexed parameter cannot be searched for directly. A good way to get an intuition for this is to look at how established codebases design their events

-   [Gnosis Safe](https://github.com/safe-global/safe-contracts/blob/main/contracts/Safe.sol)
-   [Uniswap](https://github.com/Uniswap/v2-core/blob/master/contracts/UniswapV2Pair.sol)
-   [ERC20 Specification](https://eips.ethereum.org/EIPS/eip-20)
-   [ERC721 Specification](https://eips.ethereum.org/EIPS/eip-712)

As a general rule of thumb, cryptocurrency amounts should not be indexed, and an address should be, but this rule should not be applied blindly.

### Avoid redundant events

An example of this would be adding an event when tokens are minted, because underlying libraries already emit this event.

  

## Events cannot be used in view functions

Events are state changing; they alter the state of the blockchain by storing the log. Therefore, they cannot be used in view (or pure) functions.

Events are not as useful for debugging the way console.log and print are in other languages; because events themselves are state-changing, they are not emitted if a transaction reverts.

## How many arguments can an event take?

For unindexed arguments, if you use too many arguments you will hit the stack limit fairly quickly. The following nonsensical example is valid solidity:

```solidity
contract ExampleContract {
    event Numbers(uint256, uint256, uint256, uint256, uint256, uint256, uint256, uint256);
}
```

Similarly, there is no intrinsic limit to the length of strings or arrays stored in a log.

However, there cannot be more than three indexed arguments (topics) in an event. An anonymous event can have 4 indexed arguments (we will cover this distinction later).

An argument with zero events is also valid.

## Variable names in events are optional but recommended

The following events behave identically

```solidity
event NewOwner(address newOwner);
event NewOwner(address);
```

In general, including the variable name would be ideal because the semantics behind the following example are very ambiguous

```solidity
event Trade(address,address,address,uint256,uint256);
```

We can guess that the addresses correspond to the sender, and the token addresses, while the uint256es correspond to the amounts, but this is hard to decipher.

It is conventional to capitalize the name of an event, but the compiler does not require it.

## Events can be inherited through parent contracts and interfaces

When an event is declared in a parent contract, it can be emitted by the child contract. Events are internal and cannot be modified to be private or public. Here is an example

```solidity
contract ParentContract {
  event NewNumber(uint256 number);

  function doSomething(uint256 number) public {
    emit NewNumber(number);
  }
}

contract ChildContract is ParentContract {
  function doSomethingElse(uint256 number) public {
    emit NewNumber(number);
  }
}
```

[solidity-event-inheritance.md](https://gist.github.com/RareSkills/383907606b07ff4a36c630fbe34b2e1c#file-solidity-event-inheritance-md) hosted with ❤ by [GitHub](https://github.com) | [view raw](https://gist.github.com/RareSkills/383907606b07ff4a36c630fbe34b2e1c/raw/c823d5a3d3d542ee8fcec93fb63b8fb6930c4e02/solidity-event-inheritance.md)

Similarly, events can be declared in an interface and used in the child, as in the following example.

```solidity
interface IExampleInterface {
	event Deposit(address indexed sender, uint256 amount);
}

contract ExampleContract is IExampleInterface {
	function deposit() external payable {
		emit Deposit(msg.sender, msg.value);
	}
}
```


## Event selector

The EVM (Ethereum Virtual Machine) identifies events with the keccak256 of their signature.

For solidity versions 0.8.15 or higher, you can also retrieve the selector using the .selector member.

```solidity
pragma solidity ^0.8.15;
contract ExampleContract {

    event SomeEvent(uint256 blocknum, uint256 indexed timestamp);

    function selector() external pure returns (bool) {

        // true
        return SomeEvent.selector == keccak256("SomeEvent(uint256,uint256)");
    }
}
```

The event selector is actually a topic itself (we will discuss this further in a later section).

Marking variables as indexed or not does not change the selector.

## Anonymous Events

Events can be marked as anonymous, in which case they will not have a selector. This means that client-side code cannot specifically isolate them as a subset like our earlier examples.

 ```solidity
pragma solidity ^0.8.15;
contract ExampleContract {

    event SomeEvent(uint256 blocknum, uint256 timestamp) anonymous;

    function selector() public pure returns (bool) {

        // ERROR: does not compile, anonymous events don't have selectors
        return SomeEvent.selector == keccak256("SomeEvent(uint256,uint256)");
    }
}
```

Because the event signature is used as one of the indexes, an anonymous function can have four indexed topics, since the function signature is “freed up” as one of the topics.

```solidity
contract ExampleContract {
    // valid
    event SomeEvent(uint256 indexed, uint256 indexed, address indexed, address indexed) anonymous;
}
```

Anonymous events are rarely used in practice.

## Advanced topics about events

This section describes events at the assembly level of the EVM. This section can be skipped for programmers new to [blockchain development](https://rareskills.io/web3-blockchain-bootcamps/).

### Implementation detail: Bloom filters

To retrieve every transaction that has happened with a smart contract, the Ethereum client would have to scan every block, which would be an extremely heavy I/O operation; but Ethereum uses an important optimization.

Events are stored in a [Bloom Filter](https://en.wikipedia.org/wiki/Bloom_filter) data structure for each block. A Bloom Filter is a probabilistic set that efficiently answers if a member is in the set or not. Instead of scanning the entire block, the client can ask the Bloom filter if an event was emitted in the block; it's far faster to query a Bloom filter than to scan the entire block.

This allows the client to search the blockchain much faster to find events.

Bloom Filters are probabilistic: they sometimes incorrectly return that an item is a member of the set even if it isn’t. The more members that are stored in a Bloom Filter, the higher the chance of error, and the larger the Bloom filter must be (storage wise) to compensate for this. Because of this, Ethereum doesn’t store transactions in a Bloom Filter, only events. There are far fewer events than there are transactions. This keeps the storage size on the blockchain manageable.

When the client gets a positive membership response from a Bloom filter, it must scan the block to verify the event took place. However, this will only happens for a tiny subset of blocks, so on average the Ethereum client saves a lot of computation by checking the Bloom filter for event presence first.

### Yul Events (Solidity assembly)

In the Yul intermediate representation the distinction between indexed arguments (topics) and unindexed arguments becomes clear.

The following yul functions are available for emitting events (and their EVM opcode bears the same name). The table is copied from the [yul documentation](https://docs.soliditylang.org/en/v0.8.19/yul.html) with some simplification.

| op code | Usage |
| -- | -- |
| log0(p, s) | log without topics and data mem[p…(p+s)) |
| log1(p, s, t1) | log with topic t1 and data mem[p…(p+s)) |
| log2(p, s, t1, t2) |  log with topics t1, t2 and data mem[p…(p+s)) |
| log3(p, s, t1, t2, t3) |  log with topics t1, t2, t3 and data mem[p…(p+s)) |
| log4(p, s, t1, t2, t3, t4) |  log with topics t1, t2, t3, t4 and data mem[p…(p+s)) |

A log can have up to 4 topics, but a non-anonymous solidity event can have up to 3 indexed arguments. That is because the first topic is used to store the event signature. There is no opcode or Yul function for emitting more than four topics.

The unindexed parameters are simply [ABI encoded](https://www.rareskills.io/post/abi-encoding) in the memory region \[p…(p+s)) and emitted as one long byte sequence.

Recall earlier that there was no limit in principle for how many unindexed arguments an event in Solidity can have. The underlying reason is that there is no explicit limit on how long the memory region pointed to in the first two parameters of the log op code takes. There is of course, limits provided by the contract size and memory expansion gas costs.

## Gas cost to emit Solidity event

Events are substantially cheaper than writing to storage variables. Events are not intended to be accessible by smart contracts, so the relative lack of overhead justifies a lower [gas cost](https://rareskills.io/post/gas-optimization).

The formula for how much gas an event costs is as follows ([source](https://github.com/wolflo/evm-opcodes/blob/main/gas.md#a8-log-operations)):

```
375 + 375 * num_topics + 8 * data_size + mem_expansion_cost
```

Each event costs at least 375 gas. An additional 375 is paid for each indexed parameter. A non-anonymous event has the event selector as an indexed parameter, so that cost is included most of the time. Then we pay 8 times the number of 32 byte words written to the chain. Because this region is stored in memory before being emitted, the memory expansion cost must be accounted for also.

The most significant factor in an event’s gas cost is the number of indexed events, so don’t index the variables if it isn’t necessary.

## Conclusion

Events are for clients to quickly retrieve transactions that may be of interest. Although they don’t alter smart contract functionality, they allow the programmer to specify which transactions should be quickly retrievable. This is important for improving transparency in smart contracts.

Events are relatively cheap gas-wise compared to other operations, but the most important factor in their cost is the number of indexed parameters, assuming the coder does not use an inordinate amount of memory.

## Learn More

Like what you see here? See our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp) to learn more.

We also have a free [solidity tutorial](https://www.rareskills.io/learn-solidity) to get you started.

*Originally Published April 1, 2023*