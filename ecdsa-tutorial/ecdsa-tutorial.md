# The intuition behind elliptic curve digital signatures (ECDSA)

This article explains how the ECDSA (Elliptic Curve Digital Signature Algorithm) works as well as *why* it works.

We will incrementally "rediscover" the algorithm from first principles in this tutorial.

## Prerequisites
We assume prior knowledge of
- [Elliptic Curve Arithmetic](https://www.rareskills.io/post/elliptic-curve-addition)
- [Elliptic Curve Arithmetic in Finite Fields](https://www.rareskills.io/post/elliptic-curves-finite-fields)

## Digital Signature Recap
A digital signature algorithm is a protocol which enables a user to give a cryptographic seal of approval for a string.

Such a user has a *public key* (an $(x,y)$ pair corresponding to an elliptic curve point). The point was generated from a *private key* which is a secret scalar value.

Given a public key, a message, and a *signature*, another user can verify that the signature was produced by someone who owns the private key for that public key, and only that person could have generated the signature.

For example, cryptocurrencies use digital signatures to verify a user really wanted to make a transaction. A user, Alice, can send a message to the blockchain saying "Send 10 of my tokens to Bob." The blockchain network must be certain Alice really gave her seal of approval to that message and that Bob isn't stealing Alice's coins.

The signature produced must be specific to the message and not be accepted for other messages. Otherwise an attacker could use a signature to get the victim to approve a malicious transaction. If the signature for the message "transfer 10 coins to Bob" could also be used as a seal of approval for the message "transfer 10 coins to Eve," then Eve could use the signature from the Bob transfer to steal coins from Alice.

We call the person generating the signature the *prover* or *signer* and the person testing if the (Public Key, message, signature) tuple is valid the *verifier*.

At its heart, an elliptic curve digital signature is a proof of knowledge of a private key. Specifically, it is a proof of knowledge of the discrete log of an elliptic curve point.

For readers who have already seen the ECDSA algorithm:

$$
\begin{align*}
r &= kG\\
s &= k^{-1}(h + rp)\\
r &\stackrel{?}= s^{-1}(hG + rP)
\end{align*}
$$

but don't know where the formulas came from, you will learn the thinking behind how they were derived.

## Terminology and Notation
We refer to elliptic curve points with capital letters, such as $Q$. The generator will be referred to as $G$. The value of $G$ is known to all parties involved. Multiplication between a scalar and a point is written as $aG$, which means $G$ is added to itself $a$ times.

Given two points $P$ and $Q$, we say someone knows the *discrete log relationship* between $P$ and $Q$ if they know $x$ such that $xP = Q$, or alternatively, $y$ such that $P = yQ$.

We say someone *knows the discrete log* of a point $Q$ if they know a scalar $q$ such that $Q = qG$.

We refer to the discrete log of a point with the lowercase version of the same letter unless otherwise noted.

All arithmetic is done in a [finite field](https://www.rareskills.io/post/finite-fields), but we omit the $\pmod p$ notation for brevity.

## Failed attempts at digital signatures
### Naive approach: reveal the private key
We want to prove we know the private key $p$ for the public key $P$ where $P = pG$.

The naive approach is to reveal the private key $p$ to the verifier so the verifier can check that indeed $P = pG$.

Of course, this defeats the purpose of keeping $p$ private.

### Proving knowledge of the discrete log relationship between $P$ and $Q$
Proving we know $p$ is equivalent to proving we know the discrete log relationship between the generator $G$ and the public key $P$. That is, $G$ is added to itself $p$ times to get $P$. Revealing the discrete log relationship between $G$ and $P$ reveals the private key.

Therefore, instead of proving that we know the discrete log relationship between the generator $G$ and the public key $P$, we can prove that we know the discrete log relationship between $P$ and $Q$, where $Q$ is chosen by the prover and whose discrete log $q$ is unknown to the verifier. If the discrete log of $Q$ is unknown to the verifier, then the verifier cannot derive the private key, given the discrete log relationship between $P$ and $Q$.

Let $s$ be the number of times $P$ needs to be added to itself to produce $Q$; in other words, $sP = Q$. If the prover knows the discrete log of $P$ and $Q$, as $p$ and $q$ respectively, they can easily compute $s$ as $s = q/p$.

$s = q/p$ was derived as follows:

$$
\begin{align*}
Q &= sP\\
Q &= qG\\
P &= pG\\
qG &= spG \\
q &= sp \\
\frac{q}{p} &= s\\
\end{align*}
$$

Then the verifier can check that indeed $Q = sP$, and this shows the prover knows the discrete log relationship between $P$ and $Q$.

However, this approach fails because a malicious prover can take someone else's $P$ (of which they do not know the private key), and choose a random value $s$, compute $Q = sP$, and then send $(P, Q, s)$ to the verifier.

Hence, the prover showing they know the discrete log relationship between $P$ and $Q$ is not sufficient to prove they know the discrete log of $P$ itself. It only shows they multiplied some $P$ by $s$ to produce $Q$.

### Preventing randomly chosen $Q$
The above approach failed because the prover can present $Q$ without actually knowing the discrete log of $Q$ itself.

What if, to establish that the prover knows the discrete log of $Q$, the prover reveals $q$ (the discrete log of $Q$) to the verifier?

In this case, the prover must reveal both $q$ and $s$ to the verifier. Then, $s$ proves that the prover knows how many times $P$ needs to be added to itself to get $Q$, and $q$ proves that the prover knows the discrete log of $Q$. Presenting $q$ demonstrates that $Q$ wasn't selected by choosing a value $s$ and adding $P$ to itself $s$ times.

In that case, the verifier checks that

$$
\begin{align*}
qG &\stackrel{?}{=} Q &&\text{ // prover knows discrete log of Q}\\
sP &\stackrel{?}{=} Q &&\text{ // prover knows how many times P needs to be added to itself to get Q}
\end{align*}
$$

With these checks, the prover can't create or produce a valid relation without knowing the discrete log of $Q$ and the discrete log of $P$.

However, the verifier can compute the private key based on the formula derived previously:
$$
\begin{align*}
s = \frac{q}{p}\\
p = \frac{q}{s}\\
\end{align*}
$$

So we have a conundrum: if we reveal the discrete log of $Q$, the verifier can compute the private key. If the prover can pick $Q$ at will, then the prover can create signatures for public keys they don't know the discrete log of.

We cannot have the verifier be able to compute the private key, so revealing $q$ is out of the question.

Therefore, our solution must include the prover picking $Q$ -- but we must prevent the prover from being able to pick $Q$ completely arbitrarily (such as by picking an arbitrary $s$ and producing $Q = sP$).

Going forward, the prover must prove they know the discrete log of $Q$ and the discrete log of $P$, but not reveal either of the discrete logs $p$ or $q$.

It turns out that proving we know the discrete log of *two* points without revealing them is easier than proving we know the discrete log of *one* point without revealing the single discrete log.

## If the prover knows the discrete logs of $P$ and $Q$, then what else must they know?

The key idea is this:

*If the prover knows the discrete logs of $P$ and $Q$, then not only must they know $s$ such that $sP = Q$, they must also know $s'$ such that $s'(hG + P) = Q$ where $h$ is an arbitrary value chosen by the verifier and is a value the prover cannot predict.*

In other words, we are going to start including additive and multiplicative "shifts" in the relationship between $P$ and $Q$ -- and if the prover knows the discrete logs of $P$ and $Q$, then they must be able to compute $s'$ such that $s'(hG +P) = Q$ as long as the prover knows how much the shift $h$ is.

### An additive shift prevents forgeries
To prevent the prover from choosing $Q$ such that it is a simple scalar multiplication of $P$, the verifier can inject an additive shift $h$.

After the prover sends $P$ and $Q$ to the verifier, the verifier responds with $h$ (a public scalar value). The prover must then produce $s$ such that $Q = s(hG + P)$ (we do not need the $s'$ notation anymore).

Now the prover does not control the discrete log relationship between $P$ and $Q$, they must instead show they know the discrete log relationship between $(hG + P)$ and $Q$.

Since the verifier is already in possession of $P$ and $Q$, the prover cannot just pick a random $s$ and generate $Q$ such that $Q = s(hG + P)$. The prover must prove they know the discrete log relationship between the new point $(hG + P)$ and the original $Q$.

If the prover knows the discrete logs of $P$ and $Q$, then $s$ is easy to compute as:

$$
s = \frac{q}{h + p}
$$

Specifically, the prover solved

$$\begin{align*}
s(hG + P)&=Q\\
s(h + p) &= q\\
s&=\frac{q}{h+p}
\end{align*}$$

Let's summarize the interactive algorithm we just created:

1. We assume $G$ (and the elliptic curve group it belongs to) is agreed upon by both parties.
2. The prover publishes their public key $P$ and wishes to prove they know the private key $p$.
3. The verifier responds with a scalar $h$.
4. The prover picks a random $q$ and computes $Q = qG$.
5. The prover computes $s = q(h + p)^{-1}$ and sends $(s, Q)$ to the verifier
6. The verifier checks if $Q \stackrel{?}= s(hG + P)$

The last check works because under the hood the $s$ cancels out the discrete logs of $h$ and $p$:

$$
\begin{align*}
Q &= s(hG + P)\\
qG &= \frac{q}{h + p}(hG+P)\\
qG &= \frac{q}{h + p}(hG+pG)\\
qG &= \frac{q}{h + p}(h+p)G\\
qG &= \frac{q}{\cancel{h + p}}(\cancel{h+p})G\\
qG &= qG
\end{align*}
$$

#### Defense against forged signatures
Let's see how a malicious prover could try to compute $s$ without knowing the discrete logs of $P$ and $Q$ and see why the attempt fails.

The prover invents a value $\tilde{q}$ and generates $Q = \tilde{q}P$ and sends $Q$ to the verifier. Let $\tilde{q}$ be a value invented by the malicious prover, and not the the discrete log of $Q$.

The verifier responds with scalar $h$.

The malicious prover must now solve the equation

$$s(P + hG) = Q$$

Since the prover knows $Q = \tilde{q}P$ (but doesn't know $p$) they can try two substitutions:
- $Q = \tilde{q}P$
- $P = \tilde{q}^{-1}Q$.

However, whichever way the prover substitutes $Q$ or $P$, they run into the issue that they cannot solve for $s$ because the discrete log of $P$ is unknown to them.

##### Substitute $Q$
Here the prover substitutes the $Q$:

$$s(P + hG) = \boxed{Q}$$

$$s(P + hG) = \boxed{\tilde{q}P}$$

$$s = \frac{\tilde{q}P}{P + hG}$$

We cannot divide elliptic curve points, so to compute the fraction above, we need to know $p$:

$$s = \frac{\tilde{q}p}{p + h}$$

But the prover doesn't know $p$, so they can't compute $s$.

##### Substitute $P$
Now the malicious prover tries substituting $P$ instead of $Q$ but runs into the issue that they do not know the discrete log of $Q$:

$$s(\tilde{q}^{-1}Q + hG) = Q$$

$$s = \frac{Q}{(\tilde{q}^{-1}Q + hG)}$$

Now the malicious prover needs to know the discrete log of $Q$ to compute the formula above. However, the prover does not know the discrete log of $Q$, they only know it is $\tilde{q}P$, but they do not know $p$. Therefore, the malicious prover cannot produce $s$.

### Why the shift needs to be additive
How did we know to add $hG$ to $P$ instead of multiplying $hP$ and asking the prover to come up with $s$ such that $Q = hsP$?

The problem is that with a multiplicative shift, the prover can cancel out the factors of the discrete logs of $Q$ and $P$.

As a recap: the a malicious prover does not know $q$, the discrete log of $Q$, or $p$, the discrete log of $P$. However, they do know that $Q$ is $\tilde{q}$ times larger than $P$, where $\tilde{q}$ is a number they invented and not the true discrete log of $Q$.

If the verifier presents $h$, and asks the prover to come up with an $s$ such that $Q = shP$, then the prover simply computes $s = \tilde{q}/h$.

This will pass the verifier's test:

$$
\begin{align*}
Q &= shP \\
Q &= \frac{\tilde{q}}{h}h P\\
Q &= \frac{\tilde{q}}{\cancel{h}}\cancel{h} P\\
Q &= qP \\
\end{align*}
$$

Therefore, the protocol must include an additive shift so there is no simple multiplicative relationship $sh$ between the first point ($P + hG$) and the second point $Q$.

### Making our protocol non-interactive
Unfortunately, our solution requires the verifier to send $h$ after receiving $P$ and $Q$, requiring the prover and verifier to *interact* with each other.

If we want to make our protocol non-interactive, then $h$ cannot come from the verifier and the prover must produce it.

If the prover knows $h$ in advance (which is necessary if the proof is not interactive), then we start back at our original problem.

In this scenario, a malicious prover can pick $h$ randomly then pick $s$ randomly, and compute

$$Q = s(hG + P)$$

and then send $(P, Q, s, h)$ to the verifier. The verifier doesn't know if $Q$ was generated by a random $s$.

To avoid the prover choosing $h$ and the interaction with a verifier, we can simulate the verifier picking $h$ simply by hashing $Q$:

$$h = \mathsf{hash}(Q)$$

Since the verifier will pick $h$ randomly anyway, the prover can compute $h$ in a pseudo-random way using a hash function. This technique is called the *Fiat-Shamir Transform*.

#### A functional proof of knowledge of a private key
The algorithm is as follows:

1. The prover picks a random scalar $q$ and computes $Q = qG$.
2. The prover computes $h = \mathsf{hash}(Q)$.
3. The prover computes $s = \frac{q}{h + p}$.
4. The prover sends $(P, Q, s, h)$ to the verifier.
5. The verifier computes $h = \mathsf{hash}(Q)$.
6. The verifier checks that $Q = s(hG + P)$.

The reason this is secure is because a malicious prover cannot execute step 3 without knowing the discrete log of $Q$ and the discrete log of $P$.

##### Forgery is impossible
Suppose a malicious prover picks $\tilde{q}$ and generates $Q = \tilde{q}G$. Here, the prover knows the discrete log of $Q$, but not the discrete log of $P$. After hashing $Q$ to get $h$, the malicious prover needs to compute $s$ such that $Q = s(hG + P)$. However, they cannot do this because they do not know the discrete log of $(hG + P)$, as the discrete log of $P$ is unknown to them.

Thus, we have demonstrated a secure algorithm for proving knowledge of a private key without revealing the private key.

## The pre-ECDSA algorithm
The algorithm we introduced above simply proves knowledge of a private key, it doesn't allow the signer to sign a message.

In ECDSA, a signature is a proof that we know the discrete log of the point generated by additively shifting our public key by $hG$ where $h$ is the hash of the message and the discrete log of another point $Q$.

However, this current introduces a security bug as the prover can *first* compute $h = \mathsf{hash}(\text{message})$ and then pick a random $s'$ and compute $Q = s'(hG + P)$. $Q$ is now fully under the control of the prover again, so we can't trust that the prover actually knows the discrete log $q$.

To secure this, we again need to introduce some additional pseudo-randomness that is beyond the prover's control.

### Adding randomness $r$
Let $r$ be a random value derived from hashing $Q$. If we multiply $r$ by $P$ as follows, we have a secure digital signature algorithm. At first, this appears to create a circular dependency, but it is actually core to making the algorithm secure. The algorithm for the prover showing they know the discrete log of $(rP + hG)$ and $Q$

$$Q = s(hG + rP)$$

1. The prover picks a random $p$ and publishes their public key $P$ as $P = pG$.
2. The prover picks a random scalar $q$ and computes $Q = qG$.
3. The prover picks a message string $\text{message}$ and computes $h = \mathsf{hash}(\text{message})$.
4. The prover computes $r = \mathsf{hash}(Q)$.
5. The prover computes $s = q\cdot(h + rp)^{-1}$.
6. The prover sends $(\text{message}, Q, s)$ to the verifier
7. The verifier computes $r' = \mathsf{hash}(Q)$
8. The verifier computes $h = \mathsf{hash}(\text{message})$
9. The verifier checks that $Q \stackrel{?}= s(hG + r'P)$

Even though the prover can pick an arbitrary $h$, they cannot control the discrete log of the point $(hG + rP)$ because $r$ is generated by hashing $Q$. Therefore, to compute $s$, the discrete log relationship between $Q$ and $(hG + rP)$, the prover must actually know the discrete log $p$ of $P$.

### Optimizing the pre-ECDSA algorithm
#### Optimization 1: Hashing $Q$ is unnecessary because elliptic curve multiplication behaves like a hash function already

Let's think back to what $r = \mathsf{hash}(Q)$ was trying to accomplish. The final verification formula is:

$$Q \stackrel{?}= s(hG + rP)$$

The idea is we want $r$ to be dependent on $Q$ so that $Q$ is not wholly determined by the value $s$ over which the prover has full control. In order to compute $s$, the prover has to know the discrete log of $P$.

We want a cheaper alternative to hashing that makes $r$ dependent on $Q$.

Consider that if $r$ is dependent on $Q$, then it is also dependent on the discrete log of $Q$, $q$, as $q$ entirely determines $Q$.

Now consider that if we are given a scalar value $a$, the elliptic curve point $A$ created by $A = aG$ appears random. There is no apparent relationship between $A$ and $a$. This lack of an apparent relationship is what makes computing the discrete log hard.

Hash functions have an output that appear random also: there is no apparent relationship between the input and output of a hash.

Therefore, we can treat $Q = qG$ the same way we would treat $\mathsf{hash}(q)$. That is, $Q$ itself behaves like the output of a hash. However, we cannot set $r = Q$ since they are not the same type. Instead, we can simply take the $x$ value of $Q$ and make that $r$ (remember, $Q$ is an $(x, y)$ point).

Now $r$ is essentially the hash of $q$, and since $Q$ directly depends on $q$, $r$ behaves like a hash of $Q$.

Therefore, instead of computing $r = \mathsf{hash}(Q)$, we set $r = Q.x$ (meaning the $x$ value of the point $Q$.) 

#### Optimization 2: We don't need to send the entire point $Q$, only the $x$ value of $Q$

Elliptic curve points consist of two scalars: the $x$ and $y$ value of the point. Each $x$ value has only two possible $y$ values, the solutions to $\sqrt{x^3 + b} \mod p$.

Thus, there is no need to send $Q$, only $r$ because $r$ is the $x$ value of $Q$.

## The Complete ECDSA Algorithm: Step-by-Step
We now show the standard ECDSA algorithm along with the most commonly used notation. We note where our notation differs.

1. The prover picks a random $p$ and publishes their public key $P$ as $P = pG$.
2. The prover picks a message they want to sign and hashes it to get $h = \mathsf{hash}(\text{message})$.
3. The prover picks a random scalar $k$ (what we have been calling $q$, but the literature calls it $k$) and computes $R = kG$. Again, what we called $Q$ the literature calls $R$. Only the $x$ value of $R$ is kept as $r = R.x$.
4. The prover computes $s$ for $R = s^{-1}(hG + rP)$ as

$$
s = \frac{h + rp}{k}
$$

*Note that $s$ is "inverted" compared to our previous implementation. In the actual ECDSA algorithm, the prover computes $s^{-1}$ and the verifier inverts $s$ later, as we will see.*

5. The signer sends $(P, h, r, s)$ to the verifier.
6. The verifier computes $R' = s^{-1}(hG + rP)$ and checks that the $x$ value of $R'$ is equal to $r$.

The formula works under the hood as follows:

$$\begin{align*}
&=s^{-1}(hG + rP)\\
&=\frac{k}{h + rp}(hG + rP)\\
&=\frac{k}{h + rp}(hG + rpG)\\
&=\frac{k}{h + rp}(h + rp)G\\
&=\frac{k}{\cancel{h + rp}}(\cancel{h + rp})G\\
&=kG
\end{align*}$$

This produces the same point $R$, and hence the $r$ will match the $x$ value of the point computed by $s^{-1}(hG + rP)$.

## Deriving the public key given a signature
The Ethereum and Bitcoin blockchains do not verify signatures, given the public key, message, and signature. Instead, given the message and signature, they solve for the public key, and check that the public key matches the expected one.

To see how this works, we can do a little algebra on the verification formula to derive the public key given the signature and the hash of the message.

$$
\begin{align*}
R &= s^{-1}(hG + rP)\\
R &= s^{-1}hG + s^{-1}rP\\
R - s^{-1}hG &= s^{-1}rP\\
sr^{-1}R-r^{-1}hG&=P\\
\end{align*}
$$

However, just $(\text{message}, r, s)$ is not sufficient because $r$ corresponds to two possible points for the two $y$ values of $r = R.x$. To disambiguate, the signer also needs to send a Boolean variable to indicate which value of $y$ is being used. Sending the entire $y$ would take up more space.

The ECDSA algorithm to "recover" the public key given a signature is as follows:

1. The prover publishes their public key $P$ as $P = pG$.
2. The prover picks a message they want to sign $\text{message}$ and hashes it to get $h = \mathsf{hash}(\text{message})$.
3. The prover picks a random scalar $k$ and computes $R = kG$.
4. The prover solves for $s$ in $R = s^{-1}(hG + rP)$ as $s = \frac{h + rp}{k}$.
5. The prover sends $(\text{message}, r, s, v)$ to the verifier where $v$ is a boolean indicating which $y$ value of $r$ is being used.
6. The verifier derives $R$ from $v$ and $r$.
7. The verifier derives the public key $P$ as

$$P = sr^{-1}R - r^{-1}hG$$

and accepts the signature if the computed $P$ matches the public key $P$ that the prover published.

## Vulnerabilities in ECDSA if misused
### Malleability in ECDSA
Given a signature $(\text{message}, r, s, v)$, an attacker can compute a second signature $(\text{message}, r, s', v')$ such that $v \neq v'$ and $s \neq s'$ that recovers to the same public key $P$.

The attacker simply computes the additive inverse of $s$ as $s' = s^{-1}$ and flips the value of $v$ so that $R$ becomes its additive inverse. Essentially, we have replaced $s$ with $-s$ and $R$ with $-R$. Since these two values are multiplied together, the -1s cancel out and the recovered public key is the same:

$$\begin{align*}P &= (-s)r^{-1}(-R) - r^{-1}hG\\
&= sr^{-1}R - r^{-1}hG
\end{align*}$$

To prevent this attack, the verifier should not accept a different signature for the same message they have seen before. The more generalized and robust solution is to use a nonce, which is an always increasing number the prover must include in their message. A nonce (which stands for "number used once") serves as a unique identifier for each signature. By requiring the prover to incorporate a new, larger nonce with each signature, the verifier can easily detect and reject attempts to reuse or modify previous signatures.

### If the verifier does not hash the message, then fake proofs can be created.

Let's consider that an attacker can create a value $R$ that we do not know the discrete log of, but we know the *components* of. For example, the attacker picks random values $a$ and $b$ and computess $R = aG + bP$, and $r = R.x$.

Then, the attacker computes $s = r/b$ and $h = ar / b$.
 
We can see how this attack works by plugging in the false values for $s$ and $h$ into the verification formula:

$$
\begin{align*}
R &\stackrel{?}= s^{-1}(hG + rP) \\
R &\stackrel{?}= (\frac{r}{b})^{-1}((\frac{ar}{b})G + rP) \\
R &\stackrel{?}= (\frac{b}{r})((\frac{ar}{b})G + rP) \\
R &\stackrel{?}= (aG + bP)
\end{align*}
$$

The defense against this attack is simple: $h$ must be the result of the verifier hashing the message. When a message is hashed, it is extremely unlikely that the hash would be exactly $ar/b$.

## Summary
The ECDSA algorithm is a proof of knowledge of the discrete log relationship between an arbitrary point $R$ and a point which represents the sum of the message hash multiplied by the generator and the public key, and the public key is multiplied by a pseudorandom value the signer does not control.

Specifically, it is a proof of knowledge of the discrete log relationship between $R$ and $(hG + rP)$.

Although the signer can pick $R$ arbitrarily, they cannot control the discrete log of the point $(hG + rP)$ because $r$ pseudorandomly depends on $R$. The only way the signer can compute $s$ in $R = s(hG + rP)$ is if they actually know the discrete log of $P$, i.e. the private key.

## Credits and Acknowledgements
The following resources were consulted during the creation of this article:
- https://cryptobook.nakov.com/digital-signatures/ecdsa-sign-verify-messages
- https://jimmysong.medium.com/faketoshis-nonsense-signature-8700a44536b5
