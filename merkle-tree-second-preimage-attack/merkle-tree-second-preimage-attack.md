# The second preimage attack for Merkle Trees in Solidity

The _second preimage attack_ in Merkle trees can happen when an intermediate node in a Merkle tree is presented as a leaf.

The name of this attack is quite misleading because it implies hashes have a second preimage. Modern hash functions do not have multiple (computable) preimages.

A better name for this attack would be “node as leaf attack” or “shortened proof attack.”

## Prerequisites

We assume the reader is familiar with Merkle trees and Merkle proofs.

## Notation

We refer to `h(x)` to be the hash of x. `h(x + y)` is the hash of the concatenation of `x` and `y`. In this article, we’ll focus on keccak256 as our hash function; picking a specific function will help make some of the reasoning clearer later in the article. However, keep in mind that the ideas in this article will apply for any hash function. We refer to the leaves of a Merkle tree with ℓ. The _ith_ leaf is referred to with ℓᵢ.

## An Example Attack

Suppose we wish to create a proof for leaf 2 (`ℓ₂`) in the tree below. The leaf will be `ℓ₂`, and the proof will be `\[h(ℓ₁), h(b), h(f)\]`. The values a, b, c, …, g are the concatenation of the child values. We accept the proof if `h(g)` equals the Merkle root.

![Merkle tree with Merkle proof](https://static.wixstatic.com/media/935a00_c3c84938cff54f0f93b905ecbc65ca91~mv2.jpg/v1/fill/w_740,h_555,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_c3c84938cff54f0f93b905ecbc65ca91~mv2.jpg)

The proof will be `\[h(ℓ₁), h(b), h(f)\]` which are marked in green. The proof to the root is

```
h(a) = h(h(ℓ₁) + h(ℓ₂))
h(e) = h(h(a) + h(b))
h(g) = h(h(e) + h(f))
return root == h(g)
```

### The second preimage attack

What if the attacker provides a as a leaf and `\[h(b), h(f)\]` as the proof?

![second preimage attack](https://static.wixstatic.com/media/935a00_1d68bb1b01e04adfbf7a343850df7c75~mv2.jpg/v1/fill/w_740,h_555,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_1d68bb1b01e04adfbf7a343850df7c75~mv2.jpg)

The contract will see `a` as a leaf, where `a = h(ℓ₁), h(ℓ₂))`. If the proof is `[h(b), h(f)]`, the Merkle proof will be accepted as valid.

Essentially, if a Merkle proof is valid, then a shortened version of it is also valid if we pass the first value in the original proof as a leaf.

**Therefore, the attacker will have provided a “leaf” and a proof the contract accepts, but the “leaf” given is not a leaf in the original Merkle tree!**

So how to we prevent this from happening?

## The OpenZeppelin warning

In the OpenZeppelin Merkle tree library, we see the warning in the comments along with some solutions. We explain their solutions in the following sections.

![openZeppelin warnings](https://static.wixstatic.com/media/935a00_f4837d828df44f6f923b590367d7119e~mv2.png/v1/fill/w_740,h_477,al_c,q_85,usm_0.66_1.00_0.01,enc_auto/935a00_f4837d828df44f6f923b590367d7119e~mv2.png)

The following two sections explain how these two solutions prevent the issue.

## The attack requires 64 byte leaves

For this attack to work, the attacker must pass in the preimage of the intermediate node, not its hash value. This means it must pass `a` as the leaf, not `hash(a)`. Since Solidity uses 32 byte hashes, `a = h(ℓ₁) + h(ℓ₂)` will be 64 bytes long.

If the contract does not accept 64 byte leaves as input, then the attack will not work.

That is, if the input to the leaf has a length other than 64 bytes, it is impossible to pass `h(ℓ₁) + h(ℓ₂)` as a false leaf value.

## Using a different hash for the leaf as a defense

If our application need to accept 64 byte leaves for some reason, then we can prevent the attack by using a different hash for the leaves than for the proof.

That is, when the leaf is first hashed, we used a different hash than what we use for hashing to the root. This will prevent the attacker from “reconstructing” an intermediate node as if it were a leaf. Using the diagram above, the attacker is using `h(a)` to create the false “leaf.” However, if leaves are passed through `h’(a)`, then the intermediate value cannot be reconstructed.

### OpenZeppelin uses a double hash as a "different hash"

Instead of using a different hash function (such as via a [precompile](https://www.rareskills.io/post/solidity-precompiles)) which would cost more gas, Openzeppelin simply hashes the leaf twice. That is, `h’(x) = h(h(x))`.

We've used <span style="color:Green">green underlines</span> to show where the hash is taken twice to construct the leaf node from the underlying data

![double hash openzeppelin](https://static.wixstatic.com/media/935a00_e62a00f0a24e4dd887b3a556d778a26c~mv2.png/v1/fill/w_740,h_561,al_c,q_90,usm_0.66_1.00_0.01,enc_auto/935a00_e62a00f0a24e4dd887b3a556d778a26c~mv2.png)

Note that the _library_ does not force you to apply the hash twice — in fact it does not force you to hash the leaf at all! Not hashing the leaf can open up attack vectors beyond the second preimage attack — see the practice problem at the end.

## Conclusion

The second preimage attack works because we can supply a shorted version of a valid proof and recreate the Merkle root. Merkle roots do have one preimage — however, the root is created in increments — not by hashing the entire proof all at once. By starting the proof at a later point in the sequence, we can have an alternative input that generates the same root.

The attack is easy to defend against — simply do not allow non-leaf values to be interpreted as leafs.

## Practice Problems

The following Capture the Flag exercise uses Merkle Proofs improperly and thus can be hacked

[RareSkills Riddles: Furry Fox Friends](https://github.com/RareSkills/solidity-riddles/blob/main/contracts/FurryFoxFriends.sol)

## Learn more with RareSkills

Please see our [blockchain bootcamp](https://www.rareskills.io/web3-blockchain-bootcamps) to see our educational offerings.

*Originally Published November 24, 2023*
