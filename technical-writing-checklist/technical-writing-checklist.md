# Checklist for Technical Writing

![An image of a completed checklist and the text ""](https://static.wixstatic.com/media/935a00_3c9bf082e0594f43857063f3602c14b5~mv2.jpg/v1/fill/w_740,h_416,al_c,q_80,usm_0.66_1.00_0.01,enc_auto/935a00_3c9bf082e0594f43857063f3602c14b5~mv2.jpg)

## Fluff

-   Are fluff transitions removed? (”It is important to note, ”Why did they do this?”, “Here’s how we can solve this problem.”) Sometimes, they are necessary for flow, but most can be removed. “It is important to note” is the most overused phrase in technical writing. Don’t use it.
-   If something is important, explain why it is important. Your explanation should be compelling enough to make it _obvious_ that the topic is important.
-   Is there any trivial information that can be removed?
-   Are there any repeated sentences that do not emphasize core ideas? It’s okay to repeat core ideas from different angles, but non-core ideas should not be repeated.
-   Any fluff words or fluff sentences?
-   Avoid writing anything that sounds promotional. Generally, the following words and phrases should not be used because they are subjective and do not convey measurable, useful, or actionable information:
    -   "novel"
    -   "innovative"
    -   "unleash"
    -   "breakthrough"
    -   "game-changing"
    -   "pioneering"
    -   "disruptive"
    -   "scalable"
    -   "empowering"
    -   "adoption"
    -   "opens up possibilities"
    -   "unparalleled"
    -   "creates opportunities"
    -   "transformative"
    -   "unveiling"
    -   "visionary"
    -   “groundbreaking”
    -   "has potential"
    -   "spearhead"
    -   "esteemed"
    -   "ambitious"
    -   "thriving"
    -   "cutting-edge"
-   Eliminate sales-y language. It is a waste of the reader's time.

## Compelling Examples

-   Could the article benefit from helpful and compelling examples? A compelling example explains the algorithm without showing the steps. For example: `divWadUp(2,3) → 0.66667, divWadDown(2,3) → 0.66666`
-   Add real-world examples where possible.
-   Provide historical context to design decisions where feasible.
-   Abstract concepts require a lot of examples to explain clearly. Provide enough examples so the reader can form a pattern in their mind.

## Improved wording

-   Could a sentence be shortened without losing information?
-   Can a sentence be split in two?
-   Can the number of clauses in the sentence be reduced?

## Providing context for facts

-   Is information motivated (i.e. why should I care?)?
-   Are there very similar facts the writer can draw an analogy to?
-   Does the writer show how a presented fact is related to the topic at hand?

## Citations and other tutorials

-   Does the article have at least one unique insight, or are you just rehashing what is already out there? It’s okay to summarize other people’s work for your own personal benefit, but it doesn’t help other people. **The more you struggle with an under-documented problem, the more other readers will thank you.**
-   If the article borrows another explanation or example, does it cite it?

## Article flow and headers

-   Does the introduction tell the reader what to expect from the article?
-   Are the information dependencies topologically sorted? That is, any discussions that assume prior knowledge should come after the prior knowledge is explained.
-   Does referring to code or past information require readers to scroll up and down? Can readers digest the essay without scrolling up on a large screen?
-   If you have a circular dependency in topics discussed, explicitly state where you expect the reader to be confused and tell them to wait for a later section.
-   If readers only skimmed the headers, would they still get a good idea of what is happening? Bad headers include “Example”, “Line 20”
-   Do the headers follow a hierarchy?
-   Are there any topic transitions that lack a header?
-   Are paragraphs split up appropriately, with no walls of text?

## Target audience

-   Does the level of complexity/reader level stay consistent?
-   Don’t assume the reader needs to be taught reentrancy one moment, then expect them to understand why Ethereum added the Blake hash [precompile](https://www.rareskills.io/post/solidity-precompiles) the next. If you aren’t sure, try to write to your past self at a snapshot in time, i.e. 3 months ago, 6 months ago, a year ago, etc.

## Article content

-   Is off-topic information removed or deferred to another article?
-   Avoid going off on “interesting tangents” unless they are entertaining or directly useful.
-   After reading the article, does it feel like any major ideas or explanations are missing?
-   All ideas must tie together. There should not be any off-topic information or unmotivated examples. Does the article explain how every topic discussed is related to each other?

## Code blocks and math formulas

-   If applicable, is the code easy to copy and paste?
-   If the code is intended to be copied and pasted, is the reader informed of the necessary prerequisites to make the code run properly?
-   If the code is not intended to be copied and pasted, a screenshot will usually be better.
-   Are code screenshots easy to read? Less white space and large font?
-   If there is an expected output from a program execution, is there a screenshot of it?
-   Do code explanations give an intuition or symbolically execute? DO NOT EXPLAIN CODE LINE BY LINE. Try to give a reader an understanding of what is going on so that when they look at the code, "everything makes sense.”
-   If you need to refer to a line number, screenshot the code and draw on the image to call attention to it. Use this as an example: [https://www.rareskills.io/post/uniswap-v2-mint-and-burn](https://www.rareskills.io/post/uniswap-v2-mint-and-burn).

-   Do large code blocks or formulas have visual cues to point to the important parts?
-   Use ALL CAPS COMMENTS to draw attention to where you want the reader to draw attention. Better yet, use a screenshot and annotate the code.
-   Make long algebraic derivations easy to skip or skim.

## Diagrams and Images

-   A picture is a thousand words; are the right pictures included?
-   Do images have minimal empty space (these make the content hard to read on mobile)?
-   Can images be read in a predictable way (left-to-right or top-to-bottom)?
-   Use a font that is easy to read in the images, and make sure they are easily read on a mobile device.
-   Images should have alt-text when published.
-   If a body of work is describing something visual, it should be visualized.

## Minimize cognitive load

-   Minimize the amount of information the reader needs in their short-term memory — repeat past information if needed if the information is expected to be new to the reader.
-   Avoid making the reader convert currency or do math in their head. Avoid abstract currencies like “token A” and do things in dollar value.
    -   Is there anything else that can be done to reduce the cognitive load for the reader?
-   Use obvious numeric examples. It isn’t immediately obvious that 6 times 68 is 408, but it is immediately obvious that 6 times 4 is 24. It is worth fiddling around with values in examples until all the values presented are obvious numerical relations.
-   Variables and other code objects should be `code formatted`.
-   Avoid directing readers to other pages to learn a prerequisite unless you are writing a follow-on to those subjects. This forces the reader to switch context. Summarize the key ideas inside the article.
-   When referring to a folder, add a forward slash after. For example "in the `src/` directory we see that..."

## Using counterfactuals and answering questions

-   Giving examples of non-functional code can sometimes be helpful if it explains why the code does not work.
-   Could any topics benefit from explaining an alternate way to accomplish the same thing?
-   Did anything raise questions that went unanswered?
-   Do any topics have an incomplete explanation?
-   Sometimes, topics are best explained by explaining their surrounding topics rather than the topics themselves. For example, it’s hard to motivate “fixed point numbers” in isolation without also explaining floating points and integers.
-   Jump on opportunities to answer questions such as “Why is it this way and not another.”

## Terminology and Definitions

-   Avoid giving the reader too many non-trivial definitions in rapid sequence. This increases cognitive load.
-   When introducing new terminology, briefly define it, even if you will explain it in detail later. Making the reader keep a new term in their mind that they don’t understand increases cognitive load.
-   If terminology is not completely trivial, making it a subheader may be helpful. That way the reader can skim backwards and find it easily if they forgot.
-   For key terms and definitions, avoid synonyms. Use the same term over and over again so it is clear to the reader what you are referring to.

## How to give good feedback as a reviewer

-   Point out areas that aren’t clear, but where you _think_ you understand the writer, write down in your own words what you think the writer is saying. This will give the writer feedback on how the reader is interpreting their words.
-   If an example is compelling or helpful, point it out so the writer doesn’t delete it later.
-   If you understand something but struggle to (i.e. it takes several re-reads), point it out. The goal is to make things easy to read. “I struggled to understand” is not an admission of incompetence.
-   Read the article out loud. You will catch more issues this way.
-   Do not give feedback about wording or spelling early in the review process. Focus on high-level feedback first. Ideally, the first round of feedback should be so high that it does not directly comment on the document.
-   Regurgitating feedback from a tool like Grammarly or a chatbot is useless feedback except at the very end of the writing process. Focus on feedback from _you as the reader_.
-   Do a quick Google search on the topic and ask a chatbot to write an article on the same subject. If the article in question doesn’t provide any fresh insights, tell the writer. This is the most valuable feedback you can give.

## Preparing a technical article

-   Write down what your target audience already knows.
-   Write down why they want to read your article.
-   Write down the major ideas you want to cover.
-   List any existing articles on the subject and why you think your article will provide added value.
-   Write down any concepts that were particularly confusing to you or those you have explained the topic to.
-   List the images and diagrams you will create to plan your writing around them.

*Originally Published March 25, 2024*
