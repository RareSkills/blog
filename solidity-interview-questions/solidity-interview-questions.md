# Over 150 interview questions for Ethereum Developers
All of these questions can be answered in three sentences or less.

## Easy
1. What is the difference between private, internal, public, and external functions?
2. Approximately, how large can a smart contract be?
3. What is the difference between create and create2?
4. What major change with arithmetic happened with Solidity 0.8.0?
5. What special CALL is required for proxies to work?
6. How do you calculate the dollar cost of an Ethereum transaction?
7. What are the challenges of creating a random number on the blockchain?
8. What is the difference between a Dutch Auction and an English Auction?
9. What is the difference between `transfer` and `transferFrom` in ERC20?
10. Which is better to use for an address allowlist: a mapping or an array? Why?
11. Why shouldn’t tx.origin be used for authentication?
12. What hash function does Ethereum primarily use?
13. How much is 1 gwei of Ether?
14. How much is 1 wei of Ether?
15. What is the difference between `assert` and `require`?
16. What is a flash loan?
17. What is the check-effects-interaction pattern?
18. What is the minimum amount of Ether required to run a solo staking node?
19. What is the difference between `fallback` and `receive`?
20. What is reentrancy?
21. What prevents infinite loops from running forever?
22. What is the difference between `tx.origin` and `msg.sender`?
23. How do you send Ether to a contract that does not have payable functions, or a receive or fallback?
24. What is the difference between `view` and `pure`?
25. What is the difference between `transferFrom` and `safeTransferFrom` in ERC721?
26. How can an ERC1155 token be made into a non-fungible token?
27. What is access control and why is it important?
28. What does a modifier do?
29. What is the largest value a uint256 can store?
30. What is variable and fixed interest rate?

## Medium
1. What is the difference between transfer and send? Why should they not be used?
2. What is a storage collision in a proxy contract?
3. What is the difference between `abi.encode` and `abi.encodePacked`?
4. `uint8`, `uint32`, `uint64`, `uint128`, `uint256` are all valid uint sizes. Are there others?
What changed with block.timestamp before and after proof of stake?
5. What is frontrunning?
6. What is a commit-reveal scheme and when would you use it?
7. Under what circumstances could `abi.encodePacked` create a vulnerability?
8. How does Ethereum determine the BASEFEE in EIP-1559?
9. What is the difference between a cold read and a warm read?
10. How does an AMM price assets?
11. What is a function selector clash in a proxy and how does it happen?
12. What is the effect on gas of making a function `payable`?
13. What is a signature replay attack?
14. How would you design a game of rock-paper-scissors in a smart contract such that players cannot cheat?
15. What is the free memory pointer and where is it stored?
16. What function modifiers are valid for interfaces?
What is the difference between memory and calldata in a function argument?
17. Describe the three types of storage gas costs for writes.
18. Why shouldn’t upgradeable contracts use the constructor?
19. What is the difference between UUPS and the Transparent Upgradeable Proxy pattern?
20. If a contract delegatecalls an empty address or an implementation that was previously self-destructed, what happens? What if it is a low-level call instead of a delegatecall?
21. What danger do ERC777 tokens pose?
22. According to the Solidity style guide, how should functions be ordered?
23. According to the Solidity style guide, how should function modifiers be ordered?
24. What is a bonding curve?
25. How does `_safeMint` differ from `_mint` in the OpenZeppelin ERC721 implementation?
What keywords are provided in Solidity to measure time?
What is a sandwich attack?
26. If a delegatecall is made to a function that reverts, what does the delegatecall do?
27. What is a gas efficient alternative to multiplying and dividing by a power of two?
28. How large a uint can be packed with an address in one slot?
29. Which operations give a partial refund of gas?
What is ERC165 used for?
30. If a proxy makes a delegatecall to A, and A does address(this).balance, whose balance is returned, the proxy's or A?
31. What is a slippage parameter useful for?
32. What does ERC721A do to reduce mint costs? What is the tradeoff?
33. Why doesn't Solidity support floating point arithmetic?
34. What is TWAP?
35. How does Compound Finance calculate utilization?
36. If a delegatecall is made to a function that reads from an immutable variable, what will the value be?
37. What is a fee-on-transfer token?
38. What is a rebasing token?
39. In what year will a timestamp stored in a `uint32` overflow?
40. What is LTV in the context of DeFi?
41. What are aTokens and cTokens in the context of Compound Finance and AAVE?
42. Describe how to use a lending protocol to go leveraged long or leveraged short on an asset.
43. What is a perpetual protocol?

## Hard
1. How does fixed point arithmetic represent numbers?
2. What is an ERC20 approval frontrunning attack?
3. What opcode accomplishes address(this).balance?
4. How many arguments can a Solidity event have?
5. What is an anonymous Solidity event?
6. Under what circumstances can a function receive a mapping as an argument?
7. What is an inflation attack in ERC4626
8. How many storage slots does this use? `uint64[] x = [1,2,3,4,5]`? Does it differ from memory?
9. Prior to the Shanghai upgrade, under what circumstances is `returndatasize()` more efficient than `PUSH 0`?
10. Why does the compiler insert the INVALID op code into Solidity contracts?
11. What is the difference between how a custom error and a require with error string is encoded at the EVM level?
what is the kink parameter in the Compound DeFi formula?
how can the name of a function affect its gas cost, if at all?
12. What is a common vulnerability with ecrecover?
13. What is the difference between an optimistic rollup and a zk-rollup?
14. How does EIP1967 pick the storage slots, how many are there, and what do they represent?
15. How much is one Sazbo of ether?
16. What can delegatecall be used for besides use in a proxy?
17. Under what circumstances would a smart contract that works on Etheruem not work on Polygon or Optimism? (Assume no dependencies on external contracts)
18. How can a smart contract change its bytecode without changing its address?
19. What is the danger of putting msg.value inside of a loop?
Describe the calldata of a function that takes a dynamic length array of `uint128` when `uint128[1,2,3,4]` is passed as an argument
20. Why are strict inequality comparisons more gas efficient than ≤ or ≥? What extra opcode(s) are added?
21. If a proxy calls an implementation, and the implementation self-destructs in the function that gets called, what happens?
22. What is the relationship between variable scope and stack depth?
23. What is an access list transaction?
24. How can you halt an execution with the mload opcode?
25. What is a beacon in the context of proxies?
26. Why is it necessary to take a snapshot of balances before conducting a governance vote?
27. How can a transaction be executed without a user paying for gas?
28. In Solidity, without assembly, how do you get the function selector of the calldata?
29. How is an Ethereum address derived?
30. What is the metaproxy standard?
31. If a try catch makes a call to a contract that does not revert, but a revert happens inside the try block, what happens?
32. If a user calls a proxy makes a delegatecall to A, and A makes a regular call to B, from A's perspective, who is `msg.sender`? from B's perspective, who is `msg.sender`? From the proxy's perspective, who is `msg.sender`?
33. Under what circumstances do vanity addresses (leading zero addresses) save gas?
34. Why do a significant number of contract bytecodes begin with 6080604052? What does that bytecode sequence do?
35. How does Uniswap V3 determine the boundaries of liquidity intervals?
36. What is the risk-free rate?
37. When a contract calls another call via call, delegatecall, or staticcall, how is information passed between them?
38. What is the difference between `bytes` and `bytes1[]`?
39. What is the most amount of leverage that can be achieved in a borrow-swap-supply-collateral loop if the LTV is 75%? What about other LTV limits?
40. How does Curve StableSwap achieve concentrated liquidity?
41. What quirks does the Tether stablecoin contract have?
42. What is the smallest uint that will store 1 million? 1 billion? 1 trillion? 1 quadrillion?
43. What danger to uninitialized UUPS logic contracts pose?
44. What is the difference (if any) between what a contract returns if a divide-by-zero happens in Solidity or if a divide-by-zero happens in Yul?
45. Why can't `.push()` be used to append to an array in memory?

## Advanced
1. What addresses do the Ethereum precompiles live at?
2. Describe what "liquidity" is in the context of Uniswap V2 and Uniswap V3.
3. If a delegatecall is made to a contract that makes a delegatecall to another contract, who is msg.sender in the proxy, the first contract, and the second contract?
4. What is the difference between how a `uint64` and `uint256` are abi-encoded in calldata?
5. What is read-only reentrancy?
6. What are the security considerations of reading a (memory) bytes array from an untrusted smart contract call?
7. If you deploy an empty Solidity contract, what bytecode will be present on the blockchain, if any?
8. How does the EVM price memory usage?
9. What is stored in the metadata section of a smart contract?
10. What is the uncle-block attack from an MEV perspective?
11. How do you conduct a signature malleability attack?
12. Under what circumstances do addresses with leading zeros save gas and why?
13. What is the difference between `payable(msg.sender).call{value: value}("")` and `msg.sender.call{value: value}("")`?
1How many storage slots does a string take up?
14. How does the `--via-ir` functionality in the Solidity compiler work?
15. Are function modifiers called from right to left or left to right, or is it non-deterministic?
16. If you do a delegatecall to a contract and the opcode CODESIZE executes, which contract size will be returned?
17. Why is it important to ECDSA sign a hash rather than an arbitrary bytes32?
18. Describe how symbolic manipulation testing works.
19. What is the most efficient way to copy regions of memory?
20. How can you validate on-chain that another smart contract emitted an event, without using an oracle?
21. When selfdestruct is called, at what point is the Ether transferred? At what point is the smart contract's bytecode erased?
22. Under what conditions does the Openzeppelin Proxy.sol overwrite the free memory pointer? Why is it safe to do this?
23. Why did Solidity deprecate the "years" keyword?
24. What does the verbatim keyword do, and where can it be used?
25. How much gas can be forwarded in a call to another smart contract?
26. What does an int256 variable that stores -1 look like in hex?
27. What is the use of the signextend opcode?
28. Why do negative numbers in calldata cost more gas?
29. What is a zk-friendly hash function and how does it differ from a non-zk-friendly hash function?
30. What does a metaproxy do?
31. What is a nullifier in the context of zero knowledge, and what is it used for?
32. What is SECP256K1?
33. Why shouldn't you get price from `slot0` in Uniswap V3?
34. Describe how to compute the 9th root of a number on-chain in Solidity.
35. What is the danger of using return in assembly out of a Solidity function that has a modifier?
36. Without using the `%` operator, how can you determine if a number is even or odd?
37. What does `codesize()` return if called within the constructor? What about outside the constructor?

## Learn more
Take our [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp) to deepen your knowledge of Ethereum smart contract development.
