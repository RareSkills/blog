# Mastering Solidity: Master the Computer Science Fundamentals First

![put in the reps text with a punch bag](https://static.wixstatic.com/media/935a00_87e13a71ea374bacb9bf8121055be8df~mv2.jpeg/v1/fill/w_670,h_670,al_c,q_85,enc_auto/935a00_87e13a71ea374bacb9bf8121055be8df~mv2.jpeg)

## I hate computer science!

I'll spare you the traditional arguments for why you should study and practice what is considered "the fundamentals."

I know, the relationship between reversing a LinkedList and writing a secure and gas-efficient [smart contract](https://www.rareskills.io/post/solana-smart-contract-language) seems non-existent.

You've probably never needed to implement an algorithm that runs in log(n) time in production.

And even if you did, you just imported a library. And you've almost certainly haven't had to debug a compiler or add a system call to an operating system. Write a buffer for UDP packets? Heaven forbid!

But you should study these subjects anyway.

Let me tell you a story.

## The martial arts kid

When I was a kid, I remember feeling "held back" when practicing martial arts.

In Karate, we weren't allowed to spar until we could strike the punching bag with flawless technique (and block punches instinctively). In Jujitsu, we weren't allowed to grapple until we had mastered breaking falls and basic throws (takedown techniques).

Half the practice sessions were cardiovascular and footwork exercises that had no apparent relationship to fighting people.

It was frustrating. I remember feeling relieved when the instructor finally allowed us to clock our classmates in the noggin. I don't think the violence made it appealing, but rather, not being split into the beginners' group in front of everyone. Something about publicly doing exercises labeled "beginner" seems intrinsically distasteful.

In retrospect, the wisdom behind this constraint is evident: if you go straight into a fight flailing your arms, you won't learn as efficiently as drilling the fundamentals directly.

Sparring is far more effective if it leverages existing habits than developing techniques from scratch. Having to think about proper technique when another classmate is swinging at you causes the skill to develop more slowly.

Actual sparring is simply the sum of the fundamentals: good footwork, good striking, good blocking, and not running out of breath.

Drilling basics is not just a martial arts thing. Competitive chess players and go players don't spend all their time playing games. Instead, they go through problem books with specific scenarios designed to teach recurring patterns like mid-game, gambits, sacrifices, and so forth — what we call the fundamentals in this article.

## Martial arts and the art of computer science

Software engineering isn't much different.

There is a huge temptation to jump straight into the technologies that are one step removed from the final application you are trying to build.

Machine learning? Learn TensorFlow! [Blockchain](https://www.rareskills.io/)? Learn solidity!

(Educators who misleadingly promise engineers better salaries if they learn these technologies are a big part of the problem).

Jumping straight into TensorFlow or solidity will ***not*** make you an effective machine learning or blockchain engineer. You'll be dealing with too many underlying skills at once. You will be like the new martial arts student flailing your arms and getting winded after 45 seconds. You might land a hit here and there, but you will always be a limited practitioner.

Master the fundamentals first and revisit them often.

![mr miyagi wants you to learn computer science](https://static.wixstatic.com/media/935a00_e15233f095684a73afb601e9999f2c9d~mv2.jpg/v1/fill/w_740,h_416,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_e15233f095684a73afb601e9999f2c9d~mv2.jpg)

## Wax on, Wax off - Bits in, Bits out

Every problem in computer science, regardless of the domain, is a string of bits going in, some operation happening, and a string of bits going out.

Every. Single. Problem.

The interpretation of these bits is domain-specific, but they are all bit sequence transformations.

There is a science to reasoning about these bit string transformations. It's called computer science.

Let me illustrate how this abstraction applies to everything in development.

The state of a blockchain is modeled as a string of bits (everyone's balances and the state of the smart contracts). You combine that state with another bit string (a transaction), put the two into a transformation, and get a new bit string: the new state of the blockchain.

Machine learning is the same thing. Data + model (both bit strings) result in a new bit string (trained model). In that field, we semantically interpret the Data as "jpegs," the model as a "network file," and the transformation as "training," but they're still just bit strings and transformations.

Rendering a website is a transformation of JSON from the API to HTML on the webpage. Bit string in, another bit string out. These have a higher level of abstraction than JSON and HTML, but under the hood, the same bit string transformation is happening.

The number of meaningful ways to apply semantic meaning to those bit strings is well documented, and you can study them rigorously.

There are basic theorems about what can and cannot be done with those bit string transformations, regardless of domain:

- If a bit string is interpreted as computer instructions, will it result in an infinite loop or not? (Undecidable, that's the halting problem).
- If a bit string is interpreted as computer instructions, can we prove it has some equivalence to another bit string? Yes, but this is generally intractable. But if we model it as a constraint solving, we can efficiently solve some cases.
- If a bit string is modeled as conveying information, can the bit string be made smaller without any information loss? How do you know when you cannot make it smaller? (This matters for saving bandwidth!)
- If a bit string is extremely large, what claims can we make about efficiently retrieving pieces of information we care about it from relevant substrings? (Database theory and distributed systems)
- Given one bit string and another bit string, output another bit string that describes their similarity. (Search algorithms)
- Or how about if a bit string represents the state of a system? Can it be transitioned to undesirable configurations? (Hacking and state machines).

Do you get the point?

Reasoning about bit string manipulation makes you good at any specialization in computer science. You aren't actually moving 1s and 0s around; you are applying powerful abstractions on top of the interpretation of the 1s and 0s. And these abstractions are the results of decades of research by some of the most brilliant thinkers in history. You truly get to stand on the shoulders of giants.

These categories of abstraction fall into clusters known as "cryptography," "information theory," "compilers," "networks," "virtual machines," and so forth. You know, the subjects that don't seem worth studying because there is an established tool for them.

Let's go back to our martial arts analogy. You might see some black belts exhibiting a really impressive sparring match, but under the hood, it's really just footwork, striking, blocks, and not running out of breath. The same phenomena powers the impressive feats coding ninjas pull off. It's just the basics fluently executed.

Everything you do in computer science is taking a string of bits that represents something and turning it into another one that represents something else. (Or if you are a hacker, identifying inputs that result in undesirable outputs).

You get better at computer science by improving your general ability to model and transform bit strings, not by writing your 10th NFT minting website.

The more abstractions and paradigms you have at your command for bit string transformation, the more capable you will be in any field of computer science and development.

## Is Leetcode Unfair?

This may sound like I'm defending tech companies for doing their 45-minute whiteboard interviews. There is value in doing this, and I am defending the principle but not the typical implementation.

We require students to pass an easy to medium data structures and algorithms test before they can join the RareSkills [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps). The difference is that they have more than the traditional 45 minutes to do it.

I readily admit it is rather cruel for someone to have to solve data structure questions in 45 minutes with a dry-erase marker. Someone who cannot solve them **at all**, however, is not a competent programmer and should not be trusted with creating smart contracts holding millions of dollars. You'd be surprised. 30% of the applicants who take the test can't even solve one single problem (and a variant of fizz buzz is one of the questions).

I'll revisit this point later, but although data structures and algorithms are an important aspect of the foundations of computer science, maybe even the most important piece, they are not the only consequential part of computer science.

## But what about framework job requirements?

Employers generally know that fundamentals matter more than application intuitively, even if most don't verbalize it explicitly.

Someone will point out, "but Jeffrey, most job descriptions say they want two years experience with React, one year with solidity, production experience with Kubernetes, etc. That's clearly what I should be optimizing for!"

If you haven't noticed already, most job descriptions are rather pie-in-the-sky.

Let's do some basic math. Let's say there are 12 frameworks and languages that really matter. An average developer can genuinely master 4 of them in a 5-year career. The probability that the developer learned the same 4 out of the 12 the employer wants is 1 out of 495.

(That's 4/12 * 3/11 * 2/10 * 1/9 or C(12, 4))

Software job descriptions often sound like an average guy who wants to date an ivy-league educated supermodel who is also Forbes 30 under 30. Good luck. Your chances are 0.2%. To be fair, a lot of software engineers have unrealistic expectations also. Matching software engineers to jobs is as messy as traditional dating. Anyway, I digress.

Despite the low odds of people matching the job requirements, people still get matched anyway. Why does the employer hire people who don't perfectly meet the job description? (Or, for that matter, why do people date partners who don't check all the boxes? Sorry, I digress again, but it's illustrative).

It's because they can see the developer has generalized skill in computer science and can adapt it to what the company needs. (Going back to dating, people partner because they see the other person has the underlying characteristics that make a good companion, not because they have features correlated with being a good companion).

I'm not saying a company will hire someone who has never built a frontend app for a senior frontend role. But I am saying within the broad categories of what we designate, "frontend, backend, [smart contract](https://www.rareskills.io/post/smart-contract-creation-cost), infrastructure, etc.," smart employers care more about general ability than framework experience.

## A gap in the English language

It's not that job postings are misleading you; it's that the English language and the American culture that dominates software development have a gap in the ability to succinctly express…

"Generalized ability within a domain that is more specific than just being smart but more general than memorizing a bunch of facts — someone who '*gets it*' where '*it*' is a set of important meta-skills that apply to the common relationship between what needs to be solved this week and is likely to need to be solved next month… And when they solve the problem, they understand the side-effects within the domain that accompany the solution, and how those side effects might leak into other domains."

Wow, that was super wordy.

(If I were super cultured and wanted to flex it, I would say the {Japanese, French, Indian, or something else not English}-equivalent word, but alas, I am not super cultured, and even if I was, the existence of this word in another language doesn't solve the problem at hand).

We have another difficulty: words like "foundations," "first principles," and "fundamentals" connote "beginner material." In linear algebra terms, we don't have a word or colloquialism that refers to the basis vectors of a knowledge space without implying that the person studying it is a noob. (See, I didn't miss my chance to flex my linear algebra knowledge on you).

Despite the terminology gap, martial arts instructors and technical hiring managers understand the concept intuitively. But because they lack the proper language to express it, they use misleading terms like "athletic," "experienced," or "fast learner."

Okay, we do have words like "expert" and "mastery," but these are very diluted and overused by educators who claim they can bestow "expertise" on you by teaching you how to parlor tricks with whatever framework is hyped today.

Even if you have been using the requested frameworks for five years, that won't help you if all you do is blindly copy tutorials online. Your deficiency will usually get caught at the interview stage. So much for "years of experience" being the key factor, right? Employers expect you to understand how the frameworks work, not only how to interface with their APIs.

All frameworks boil down to the fundamentals. They take a bit string you provide through the API and return another bit string as a state change or the result of a query. They are doing the same kind of fundamental data transformations in the foundational domains I listed earlier.

Want to master the frameworks? You know what my next sentence is!

So study the fundamentals, not the frameworks. That's how you become someone who can learn any framework quickly — even create the frameworks yourself. And that's what the employers really want, even if the current language doesn't allow them to express the desire succinctly.

## Evergreen and High Leverage Knowledge

Because "fundamentals" sounds like "beginner" (I.e., you should go into the corner and punch that bag a hundred times before you can practice with people who have a darker belt than you), I'm going to refer to certain computer science concepts as "*Evergreen*" and "*High Leverage*": "*Evergreen*" because it doesn't go out of date and "*High Leverage*" because it accelerates the acquisition of related knowledge.

Learning another language, let's say [Rust](https://www.rareskills.io/rust-bootcamp), will certainly do you some good.

But learning how compilers work will help you learn ***any*** language faster.

Learning how CPU architectures and machine code operate will help you optimize **any** smart contract on *any* blockchain.

Note that this doesn't work in reverse. Learning another language doesn't teach you much about compilers. But learning compilers will help you with programming language acquisition. That's why it's traditionally referred to as "foundational," although I prefer to call it "high leverage" in this context.

Learning how data structures and algorithms operate will help you decipher large code bases as you can read the architecture in sensible chunks rather than variable by variable. Moreover, having been through the drill of modeling and transforming bit strings in multiple ways, good solutions will come to you faster when you code in the "real world." Finally, you'll also understand why frameworks model data the way they do, and it will help you learn new frameworks faster and stay ahead of the competition.

"But Jeffrey! It will take me a year to learn this stuff, and I'll be so out of date with blockchain by then!"

That's where the "Evergreen" part comes in!

Guess what? You'll be out of date a year from now in blockchain, even if you study the current material today. You will always be out of date. The key is to navigate expiring information better than your competitors by equipping yourself to learn faster.

The laws of the universe that govern information encoding and transformation do not go out of date.

Depth-first search is over a [century old](https://en.wikipedia.org/wiki/Depth-first_search) and is still relevant today. [Radix sort](https://en.wikipedia.org/wiki/Radix_sort) (faster than quicksort in many applications) is also a century old. The Turing machine, which underlies all theories of computation, was conceptualized 80 years ago. The assumptions cryptography relies on have not changed either. It boils down to whether the encrypted text (again, a bit string) "appears random" by formal definitions.

Computer science is just the manipulation of bit sequences. Always has been. The rules that govern it are evergreen.

![computer science has always been manipulation of bit sequences](https://static.wixstatic.com/media/935a00_0425e7b808bc40349c806ceb3c1bd360~mv2.jpg/v1/fill/w_740,h_416,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_0425e7b808bc40349c806ceb3c1bd360~mv2.jpg)

### The secret behind staying up to date

Staying up to date means quickly grokking how the latest clever guy or gal recombined existing knowledge.

Major innovations are very incremental under the hood. It's just that the major innovations incremented a couple of critical variables with highly disproportionate outcomes:

Bitcoin merely combined digital signatures with proof of work (both decades-old concepts at bitcoin's time of invention). Ethereum took Bitcoin's execution core and made it Turing complete. Chat GPT took the self-attention transformer (4 years old by that point) and made it bigger, and added some hardcoded business rules.

"Innovation" is usually just a serendipitous increment of a key variable with an asymmetric result (in other words, meticulously recombining and improving something until something different and/or good happens). If you fundamentally understand what gets incremented when a new innovation rolls out, you'll master the new technology at a speed that bewilders your contemporaries.

## Making your job application stand out

Here's a secret. Do something crazy like build a compiler from scratch or implement a nontrivial cryptography algorithm from the mathematics up. This kind of project will draw far more attention from potential employers than showing them you've built a cat classifier or the 100th NFT marketplace with a staking feature.

One project shows you know how to follow a tutorial. The other demonstrates that you can tackle difficult subjects head-on and can handle the myriad of issues that may come up but don't appear in the job description.

Believe it or not, the Solidity compiler has bugs in it. Who would an employer rather hire, an engineer who just throws up their hands when the issue comes up, or an engineer who notices the issue, digs into the compiler source code, raises an issue on GitHub, and redesigns the solidity code to circumvent the issue? The first engineer may have a dozen [DeFi](https://www.rareskills.io/defi-bootcamp) staking apps on their GitHub, but that pales in comparison to the engineer who can explore the rabbit hole to its conclusion.

Now I understand: it's very hard at the early stage to see the relationship between flipping binary trees recursively and writing efficient smart contracts.

The feeling can be like when I was learning martial arts: "Why are we doing jumping jacks when we should be kicking people in the face? The connection seems so distant!"

I can tell you that absolutely grinding the fundamentals enabled me to lead a machine learning department at a major company less than two years after being introduced to the concept and creating the only expert-level Solidity courses on Udemy less than one year after doing blockchain full-time. And whatever hype field happens next, I'll learn it faster than 98% of engineers.

Why? Because every technology is built from the ground up from the same first principles, just combined differently.

The more first principles you grasp, the faster you'll be able to learn in a new subfield of computer science, as you are no longer learning from the ground up. Instead, you are simply combining the information you already know.

## But my staff engineer is great and can't leetcode!

Fundamentals are a lot broader than just leetcode. I'm not saying that because you can move two pointers around an array, you can master any domain of software development quickly. Studying data structures and algorithms have diminishing returns after a certain point (though I would argue most engineers don't reach that inflection point).

[Leetcode](https://www.rareskills.io/post/best-50-leetcode-questions-to-start) doesn't test you on networking, language theory, information leaks, virtual machines, operating systems, compilers, and so on.
It's outside the scope of discussion, but at the high levels, soft skills matter a lot. I don't want to ignore that. But we can agree that soft skills being roughly equal, the engineer with superior technical maturity will come out ahead.

You can still gain an intuition of the fundamentals if you have a lot of good exposure to problems that force you to deal with them and can recognize patterns quickly. However, it will always be more efficient to study the foundations directly rather than learn through random chance. You might not get as lucky as your staff engineer.

## How to drill the fundamentals — not just leetcode

- Learn a useless functional programming language. It will force you to see bit strings in a different way.
- Work your way up to leetcode medium in that language
- Write a compiler from scratch
- Create a simple operating system or virtual machine from scratch or modify an existing one
- Learn complexity theory (hint: it's not only O(f(n)) analysis)
- Take a cryptography course and actually re-implement the algorithms
- If you are doing machine learning, take a proper linear algebra and statistics class

Everything in computer science is a bit string that goes into some box and comes out as another bit string. You learn that in complexity theory. All of these fields are just ways to apply abstractions and useful interpretations to the bit strings. The more abstractions you know, the more powerful a developer you will be.

## Should you learn the fundamentals first?

If you are completely new to coding (chances are you aren't if you are reading this article, but I'll include this section), I don't think learning the fundamentals *first* is strictly necessary. I see a lot of CS undergrads going through the same experiences I went through as a young martial artist. Studying material without understanding why it is important is not optimal. It's great if you have a teacher who inspires you to have confidence in mastering the first principles, but not everyone has that luxury.

I think a reasonable learning journey can look like the following. This is what very successful self-taught engineers or web2 bootcamp grad's learning journey looks like.  

Just starting off → tutorial hell → learning from doing → mastering the fundamentals.

A successful four-year degree CS student might have a learning journey that looks like this:  

Mastering the fundamentals → learning from doing → revisiting the fundamentals.

It's okay to delay learning the foundational topics if you are still struggling to do the basic things in building functional applications. But don't get comfortable building functional applications because you will get stuck there if you don't master the fundamentals.

Yes, there is such a thing as tutorial hell, and the solution to it is to take the plunge and build.

But once you've gotten out of tutorial hell, the next step is not developing more applications but coding things that develops generalizable computer science skills.

If you can't build applications, you aren't going to get paid. But if you don't have the foundations, you won't get ***hired*** for a desirable job, much less acquire a desirable **promotion**, unless you have the backbone skillset needed.

## As an interviewer

I'm not advocating that every employer should do a leetcode style test or something that quizzes the fundamentals I outlined earlier. If you are only building relatively straightforward applications, then it's reasonable only to test engineers on that ability. However, the intended audience for this piece is engineers who are seeking to advance their careers (that's what RareSkills offers). Career advancement won't come from building some variation of the same application over and over.

## Conclusion

Everything we do in software development is a glorified interpretation and transformation of bits. Data transformation is a skill that can be directly developed with deliberate training. The more fields of computer science you study, the larger the vocabulary you can bring to the problem at hand. The best engineers develop these skills intentionally. The average devs go straight to using the frameworks. Frameworks go out of date. The fundamental theorems of information encoding and manipulation do not.

The difference between you and the engineer who earns \$100,000 more than you is not a knowledge of a language or framework. It's knowledge of the components of the framework. And those components depend directly on the foundations of computer science.

Go off the beaten path.

Study and practice the intimidating subjects that seem useless.

It is not just a shortcut in the long run.

It is the *only* way.

## RareSkills

Thanks for reading this article. As you can see, I am passionate about developer education. That's why I created RareSkills. It's to teach code blockchain development the right way. So please check out our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps). We offer everything from courses for those totally new to web3 coding to courses aimed at professional solidity developers.

Half our students already have a job as a smart contract developers, so we are far more advanced than any typical bootcamp. And because we heavily emphasize the fundamentals, what you learn will be relevant in fields even outside of blockchain.

*Originally Published Feb 8, 2023*
