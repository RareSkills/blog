# Getting a smart contract audit: what you need to know

A smart contract audit is a review by blockchain security experts to ensure that users will not lose funds due to a malfunction or security vulnerability. Furthermore, an audit tries to anticipate unexpected and undesirable smart contract behavior before the contract is deployed.

Navigating this field is tricky. There are dozens of audit firms, and getting quotes from several of them takes time, and it is hard to know if you are getting a fair price or not.

This article will help serve as a starting point for navigating the task of getting a smart contract audit.

References to auditing firms in this article should not be construed as an endorsement.

## There is no official standard for a smart contract audit

**And there probably won't be one for the foreseeable future with how fast the industry moves.**  

The term "smart contract audit" is an evolution of industry lingo, it does not have a rigorous definition, and it can mean different things to different people.

For those coming from industries where [ISO standards](https://www.iso.org/standards.html) are the norm, smart contract auditing will seem like the wild west (and frankly, that would be an accurate assessment).

Although some [smart contract vulnerabilities](https://rareskills.io/post/smart-contract-security) are well known, others are emergent properties of complex systems and are hard to detect systematically.

Standards are characteristic of mature markets, and [web3](https://rareskills.io/web3-blockchain-bootcamps) as a whole is not a mature market.

It is a bit of a conflation of terms to call smart contract audits "audits." In the field of accounting, an "audit" has a very rigorous definition; this is not the case in blockchain engineering. What is called an "audit" in the blockchain is a security review by a team of experts.

What makes this even more confusing is that there is no formal definition or certification for what makes someone an auditor or security expert in smart contracts. Yes, there are companies selling certifications, but these certifications hold no weight in the industry.

As you can expect, there are several enterprising individuals capitalizing on the confusion to sell expertise they don't have — as there is no established way to measure expertise.

Caveat emptor.

All that said, it is widely agreed in the industry that saying "this smart contract has been audited" means "a group of security experts was paid to review this codebase and find issues." Since there is no formal definition for expertise, industry reputation is often used as a proxy.

## Three main types of auditors

### Smart Contract Audit Companies
These are firms that specialize in security research. Some only audit smart contracts, while others audit smart contracts and traditional computer applications. You can think of this as an agency composed of security researchers. Until recently, this was the standard way to get a smart contract audit. If you do a Google search for smart contract auditing companies, these are the sort of results you will typically get.

### Independent security researchers

Rather than join a company, some security researchers work for themselves. They are a little harder to find as they have much smaller marketing budgets compared to agencies, but because they have less overhead, they can often provide better prices.  

One drawback is that a firm is able to have several auditors review your code, whereas an independent auditor will only have one reviewer. But for smaller or less complicated codebases, this might be more cost-effective.

### Competitive audits

Recently, a couple platforms ([Code4rena](https://code4rena.com/), [Sherlock DeFi](https://www.sherlock.xyz/)) have popped up where companies requesting an audit publish their source code on the platform along with a monetary prize pool, and then independent contestants try to find bugs and vulnerabilities.

The more bugs a contestant finds, the more money they will earn, but if another contestant finds the same bug, then the payout is diluted between the two. For well-understood bugs, the payout can be diluted considerably.

Earnings for auditors in this format are quite sub-par. Contestants can rarely expect to make much more than \$1,000 per contest despite putting in considerable effort. The primary incentive for participants is to prove their abilities so they can get a job at a traditional audit firm or attract business as a solo auditor (independent researcher). There are also no guarantees about the quality of the people participating in the contest, as anyone can participate.

However, these platforms can be very educational for new auditors. Because the format encourages an intense focus on finding bugs, then gives feedback on what bugs were missed, they can provide a relatively tight feedback loop which helps speed up learning. In fact, we guide some of our [solidity bootcamp](https://rareskills.io/courses/solidity-bootcamp) to participate in these contests for educational purposes.

The best auditors don't spend a significant amount of time in contests, as they earn far more money working for themselves or for a firm. So keep in mind that someone doing well in an auditing contest may simply mean that they did not face significant competition for the contests they participated in. There is also an element of luck where a single contest makes up a significant portion of an auditor's public earnings. However, this format get far more eyes on the code than other audit formats, and audit contests have caught bugs missed by other auditors. They also serve as a channel for companies to subtly market their protocols to the developer and auditor communities, so they are a valuable contribution to the ecosystem.

### Bonus: Informal code roasting

Some developer discord groups will review your code for free (out of the desire to be helpful, so please don't abuse others' generosity). If you have a small and simple codebase, sometimes getting a few experts to donate their time may be the most cost-effective route for you. It would be extremely disingenuous for such a review to be called an "audit," but if it is the only option available, it's certainly better than no review at all.

## Things to watch out for in smart contract audits

### Look at past audits from the firm

Some auditors use a rigid checklist for a list of smart contract vulnerabilities and check if the issue is present or not. This is good as a first pass, but many bugs are application-specific (as we note in that article). You want to make sure the auditor will consider the nuances and business requirements of your application. By looking at their past audits, you can see if they mechanically go against a list or dive deep into the specifics of your application. An auditor not supplying past audits is generally a red flag.

### Auditor specialization

If your project uses exotic DeFi tokenomics or [Zero Knowledge Proofs](https://rareskills.io/courses/zk-bootcamp), then you will want to make sure that your selected auditor has experience with that. Similarly, if you are building a dapp on a non-evm compatible chain like [Solana](https://rareskills.io/solana-tutorial), then you'll want an auditor experienced with that blockchain.

### Minimum cost

Some audit firms will not bother with small codebases like an ERC20 token or a basic NFT because they won't make enough money to pay for the sales call and account management.

### Rubber stamp auditors

Because there is no formal or accepted definition for an audit, some firms have popped up that produce what is derisively known as "rubber stamp audits." These firms provide audit reports simply because the buyer wanted an audit so they could tell their customers, "look, we've been audited!"

Not every protocol intentionally gets rubber stamp audits; they may simply have chosen the auditor unaware that they are rubber stamp auditors.

### How reliable are smart contract audits?

Even if a smart contract is audited, it can still get hacked afterward; this is not a rare occurrence.

[Rekt.news](https://rekt.news/) has a [leaderboard](https://rekt.news/leaderboard/) that tracks smart contract hacks by the amount of money lost. It's easy to see most of the hacked protocols were not audited at all.  

However, some of the protocols did have audits, some more than one, that the auditors still missed. In some cases, this can be because the auditors were incompetent or didn't try hard enough, but in general, it is not possible to guarantee a codebase is free of bugs.

The good news is some vulnerabilities don't only get missed by auditors; they get missed by hackers too. Some smart contracts had gone on for months with a live vulnerability before they got patched. A decent audit will make sure the "low-hanging fruit" for hackers is eliminated.

### Should I get multiple audits?

This is standard practice with well-funded DeFi protocols that intend to hold a lot of money; having three or more audit firms review the code is quite normal.  

What one auditor misses, another may catch, and vice versa. Similarly, one auditor may notice a vulnerability, and the developers may patch that vulnerability while introducing a different one.

## Typical Workflow of getting an audit

**1. Get quotes**   
You'll reach out to various firms asking for quotes, and you'll coordinate based on their availability and your launch dates.

**2. Audit begins**   
You should be in a code freeze when you go to audit. If the auditors are reviewing a different codebase from the one you will launch to prod, you are probably wasting money!

**3. Audit report**   
Most audit reports (that don't follow a fixed checklist) will return a list of findings categorized as Critical, High, Medium, Low, Informational, and Gas Optimization. These are explained in the following section. An audit can be as short as a few days to as long as a few months, depending on the scope of the project.

**4. Review fixes**   
Depending on the agreement, the auditor will review the changes made by the developers and ensure the fix actually addresses the bug. The number of revision reviews considered acceptable depends on the auditor.

**5. Publish Report**   
Auditors generally publish their report if the customer permits (and customers usually want the audit to be public to show they've been audited).

## Audit Finding Severity

Most audit reports group security findings into severity:

- Critical
- High
- Medium
- Low
- Informational
- Gas Optimization
    
Not all firms use the same labels. As we noted, with the lack of universal standards, there isn't a uniform definition for what is considered a "high" or "medium" vulnerability. Some auditors disagree about the appropriate rating to assign the same bug, but here are some factors auditors use to assign severity:

- Worst case outcome. All of the funds getting stolen is catastrophic. Bugs where a hacker cannot steal funds but users aren't adequately protected from shooting themselves in the foot would be labelled with low severity.
- Number of users affected. The entire protocol losing money is more severe than an individual user losing money.
- Incentive to conduct the attack. If an attacker needs to spend more capital to conduct an attack than they would gain for carrying it out, then the severity is reduced. This is still a vulnerability because the attacker could have a non-economic motive to carry out an attack (showing off, personal vendetta, etc).
- Obscurity of the attack. If the vulnerability is something like missing an access control, then an unsophisticated attacker can carry out the attack. If the weakness depends on a deep understanding of a sophisticated tokenomics mechanism using advanced mathematics, then it is less likely the attacker will find the issue.
    
Some auditors combine the last two factors into one: the likelihood of an attack. Under this model, they use the following table to assign severity:

|                   | **high damage**    | **medium damage** | **low damage** |
|-------------------|----------------|---------------|------------|
| High likelihood   | High/Critical  | High          | Medium     |
| Medium Likelihood | High           | Medium        | Low        |
| Low Likelihood    | Medium         | Low           | Low        |

### Critical vs High vs Medium

Some auditors make a distinction between critical vulnerabilities and high vulnerabilities, and other auditors group critical vulnerabilities with high vulnerabilities. In either case, these are considered to be extremely serious bugs! For those that make a distinction, Critical may mean the entire protocol is severely affected, whereas a High means individual users may be severely affected.

Medium severity issues usually mean issues which can be damaging but unlikely to occur.

### Informational  
Some auditors will point out issues, such as not adhering to the Solidity style guide or having variable and function names that are misspelled or misleading. These are not security vulnerabilities but could cause confusion down the road. Not all auditors point out informational issues.

### Gas Optimization  
Some auditors will suggest ways to improve the [optimize the gas cost](https://rareskills.io/post/gas-optimization) the smart contract. These are not security related, but they will certainly improve user experience.

## How much does it cost to audit a smart contract?

**Estimates based on public information**   
If you look on Fiverr, you'll find some services to audit smart contracts for low prices, but would you really want to depend on that kind of service for something this critical? Because contracts usually have non-disclosure agreements, rates can be hard to determine, but here is some public information.

**1. Reward pools for competitive audits**   
Competitive audit platforms advertise the reward pool, and of course, this money is coming from the customer paying for the audit. Here you can see prize pools as low as 5k and sometimes up to hundreds of thousands of dollars for very large projects. In the extreme case, prize pools of over 1 million dollars have been done. Of course, the people running the contest expect a fee for running the platform, so the actual cost will be higher.

**2. Contractor salaries**   
The auditing firm Spearbit published the [rates](https://twitter.com/spearbitdao/status/1664642186405728256) at which they pay their auditors (who are contractors, not employees), so one can infer how much an audit will cost (not including the operations and profit margin) based on this. The pay per auditor is 3k-20k per week depending on seniority. Keep in mind this is not a full time job, but per-engagement.

**3. Independent auditors**   
Another data point we have is independent auditors who have been public about earnings. One claims to earn [\$40k per month](https://twitter.com/pashovkrum/status/1653698582787096578) on good months, taking on about four audits per month, another claimed [\$50,000 in one month](https://twitter.com/bytes032/status/1667868533257289730) with five audits. This is on the high end of what an independent auditor can expect to make, but it's not outside the realm of possibility; staff engineers in top Silicon Valley companies earn more than that.

## How auditors generate quotes

There is no standard pricing model, but here are some practices auditors use to determine the price they quote.

**Dollars per lines of code**   
The most common pricing model is \$ / lines of code. This could cost between a few dollars to tens of dollars per line.

**Size and complexity**   
Other auditors use a combination of size and complexity. A 1,000 line NFT is considerably simpler than a 1,000 line mathematics library. The auditing firm will make a subjective judgement based on the combination of these factors.

**Similarity to existing protocols**   
Some auditors will give a discount if the project to be audited is forked, or very similar, to an established project as they can leverage past audits to determine if your project learned from past mistakes. Also, the total number of lines that have not been reviewed is a small percentage of the project. Different auditors will have a different threshold for what constitutes "similar" to established projects.

**Pay per vulnerability**   
This model is common among independent auditors. The security researcher is paid a small downpayment and receives a payout for each bug found with a higher price for higher severity issues. This has the advantage of incentivizing the auditor to search harder, but it can also incentivize exaggerating the severity of bugs.

## Why are audits so expensive?

Compared to other highly specialized knowledge workers, like lawyers (the best of whom can pull in over [one million dollars per month](https://www.wsj.com/articles/on-wall-street-lawyers-make-more-than-bankers-now-ae8070a7)), smart contract audits are not prohibitively expensive, but they can be a significant burden for small companies.  
Becoming a smart contract auditor isn't easy. Generally, the skill level required is comparable to trying to get a highly competitive job in Silicon Valley, which means starting salaries are going to easily be in six-figure territory. An auditing firm needs to charge a significant margin on top of those salaries to stay profitable, so it shouldn't be surprising that audits cost tens of thousands of dollars, if not hundreds of thousands of dollars.

## Test your code first!

A shocking number of smart contracts get sent for audits with inadequate [unit testing](https://rareskills.io/post/foundry-testing-solidity) in place. Having silly bugs saps up auditor time from finding more serious ones, so use every testing and smart contract security tool at your disposal to make sure your contract is bug free.

## Want help from RareSkills?

RareSkills is not an auditing firm, but many of our instructors are professional auditors, and we do engage in security research. We have an insider view of which firms are reputable and which are not. Feel free to [contact us](mailto:contact@rareskills.io) and tell us about your needs, and we will refer you to the right resource based on your situation.  

If you are seeking to train your engineers in how to develop secure smart contracts, then our industry-leading [smart contract bootcamp](https://rareskills.io/courses/solidity-bootcamp) for professional engineers may be of interest to you.

*Originally Published Jun 24, 2023*