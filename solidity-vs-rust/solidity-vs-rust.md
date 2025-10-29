# Solidity vs Rust

A common misconception is that learning blockchain is about learning a new programming language. It isn't. Learning blockchain is far more akin to learning a framework than a programming language.

For example, just because you can program in Python or Go, it doesn't automatically make you a frontend developer any more than learning JavaScript or TypeScript makes you a frontend developer. To be a backend developer, you need to know frameworks such as Express.js, React, Java Spring, etc.

Let's get straight to the point.

Even if you know you want to develop on a blockchain that uses Rust (NEAR, Solana, Elrond, and so forth), you should still learn Solidity first.

I would hazard to say _if you already know Rust and you want to develop smart contracts on non-EVM blockchains, you should still learn Solidity anyway._

Solidity is very easy to learn. It looks similar to JavaScript. There are only ten or so frequently used keywords and types that might seem unfamiliar. A reasonably experienced developer can become comfortable in Solidity over a weekend with a good [Solidity tutorial](https://www.rareskills.io/learn-solidity) and long distraction-free study sessions. Every student in RareSkills becomes proficient in it after a week. Learning the language is the easy part.

Solidity has its own weirdness that throws developers off, but many of the surprising features come from leaking unexpected abstractions from the blockchain environment.

Learning the blockchain environment is where the real learning is, not the language.

Here are common stumbling blocks for developers transitioning to blockchain.

-   Function calls that move money around feel unwieldy at first.

-   Hash maps don't behave the way you expect them to (true for both Ethereum and Solana)

-   Although a blockchain is a database, interacting with it to store data persistently is unlike using any other database you may have had experience with.

-   Some functions behave like remote procedure calls even though they all look identical to regular functions.

-   You cannot take computing power for granted. Computational costs add up fast, even on blockchains that advertise themselves as computationally powerful.

-   Developers assume (rightly) that functions are protected by default in web2 applications because they can only be accessed through an API layer if the middleware isn't there to connect them. But the distinction between a function and an API is very blurry in the blockchain setting, so access controls behave differently.

-   Thinking in terms of computational costs feels unnatural to most developers. It can seem strange how a seemingly trivial re-arrangement of code can lead to considerable changes in execution cost.

-   Developers are constantly surprised to learn an NFT is just a hashmap of owners and token ids. It takes some adjustment to accept that tokens live in a smart contract, not a wallet (for most blockchains).

The above is a partial list. If you look at the courses we offer, we generally expect students to learn EVM chains (Ethereum, Avalanche, Polygon, etc.) before tackling Solana. This learning order isn't because it takes four months to learn Solidity; far from it.

It's because learning a new paradigm takes a while.

If you've had the experience as a backend developer trying to hack a frontend web app together, then you have an idea of what it's like to program in blockchain for the first time. It's not that you had a hard time understanding JavaScript, it's the framework that was hard.

## Transfer Learning for Developers

Blockchain is dissimilar to nearly every other framework most developers are familiar with. Cloud computing? Different. Frontend frameworks? Nothing in common. Backend frameworks. Nope. Mobile app development? Give me a break! Systems programming? Kind of, but not quite.

Once you know one framework, it's easier to pick up another in the same category. For example, if you are learning frontend for the first time, Vue js won't be easy, but if you already know React js, Vue will be a relative walk in the park. The same could be said of Ruby on Rails and Django.

Therefore, since learning an entirely new framework is challenging, it can be easier to take a detour to learn a simple framework that has many helpful online tutorials. Once you get used to the new paradigm and master the framework, you can move on to the one you actually care to learn.

If this were an article about "React Native vs. Flutter," I would say, "flip a coin; it doesn't matter. Both have great resources." But there is currently a wide disparity in learning resources between Ethereum and every other smart contract chain. Even though Ethereum has a wide selection of beginner-friendly tutorials, it still lacks educational resources for more advanced topics. This shortage should indicate that learning advanced topics in up-and-coming blockchains will not be smooth sailing!

When you are trying to get comfortable in this new paradigm, having a lot of resources on Stack Overflow to copy and online tutorials to find on Google makes the learning process faster.

Spending hours searching for simple answers to straightforward questions prolongs the learning process unnecessarily.

## Learning Rust for Blockchain

Rust is a famously tricky language because of odd (but useful) concepts like ownership, borrowing, and variable lifetime. Now add on Rust's take on async and concurrency, and you have a beast of a language to master.

Luckily, the subset of Rust used by blockchains is very small. My rough guess is that only about 25-30% of Rust is actually used by blockchain, and about 10-15% is used heavily. So if you nail the 10-15% that matters, Rust won't be your blocker regarding web3 development.

Caveat: this applies to [**_smart contracts_**](https://www.rareskills.io/post/smart-contract-creation-cost) (or **programs**, as Solana calls them). If you are building a blockchain client or smart contract tooling in Rust, you will need to be even more fluent in the language.

How much Rust do you need to know? If you can comfortably solve the LeetCode easy problems in Rust, you'll be able to read Solana programs without excessive difficulty. When you encounter a knowledge gap, you'll know what keywords to put into Google. How you go about this is up to you, but I recommend reading selective chapters from the book [Programming Rust](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) to give you an outline. But the usual caveat applies: don't get stuck in tutorial hell. You learn by doing, not by studying, reading, or watching videos.

In general, it takes about a month of part-time study to achieve the aforementioned level of proficiency with Rust.

But even after those 30 days of study, you still know nothing about the blockchain paradigm. That time could have been spent becoming familiar with blockchain itself.

Whatever you do, don't try to learn blockchain and Rust simultaneously. You'll have two unfamiliar concepts to navigate and have difficulty deciding what to put into Google.

Moreover, you will feel like this iconic dog:

![A dog at a computer](https://static.wixstatic.com/media/935a00_0b3a46d207454b1397ca7a8d54842f28~mv2.png/v1/fill/w_637,h_406,al_c,lg_1,q_85/935a00_0b3a46d207454b1397ca7a8d54842f28~mv2.png)


## Caveat

No law of the universe (or even a social convention) says you must learn Solidity before learning Rust. If the obstacles listed above don't deter you, by all means, don't let me tell you how to live your life. On the other hand, there is nothing wrong with learning Rust first if your heart so desires. Who am I to judge? You'll come out as a better developer, whichever path you choose.

However, you will be better off with well-argued opinionated advice rather than wishy-washy advice that tries to avoid offending anyone.

If you have to ask, learn Solidity first. Learn more about our [Rust and Solana Bootcamp](https://www.rareskills.io/rust-bootcamp) and [Solidity Bootcamp](https://www.rareskills.io/solidity-bootcamp)

*Originally published November 24, 2022*
