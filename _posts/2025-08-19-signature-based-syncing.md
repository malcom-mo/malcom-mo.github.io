---
layout: post
title: Towards More General Signature-Based Syncing
date: 2025-08-19
---

This post outlines a preliminary follow-up research idea for [our light client protocol]({% link _posts/2025-07-29-light-client.md %}).
Let's see if the signature-based design can be taken further! :thinking:


## The problem
In the [paper](https://arxiv.org/abs/2410.03347) and in [the prior post]({% link _posts/2025-07-29-light-client.md %}), we made the simplifying assumption that there is a long-running static subset of validators in the committee.
That is, there is a guaranteed quorum that is authorized to (and does) sign a long run of consecutive epoch transitions.
As such, the protocol based on transitive signatures cannot support fully dynamic committees without such a static quorum.

But what if this assumption doesn‘t hold, like for Ethereum’s sync committees?
Can we have a similarly simple signature-based protocol for *arbitrary cross-committee syncing*?


## Reducing scope
This seems like a tough problem, so let’s first go to a reduced special case where committees have size 1.
That is, we are now looking at standard certificate chains.
For example, Alice may sign Bob’s public key, Bob may sign Charlie’s and Charlie may sign Davy’s so that a verifier who only knows and trusts Alice’s public key can walk along the chain and have confidence in any Davy-signature.
If the chain grows to length $m$, the verifier needs to download all $m$ intermediary signatures and public keys.
So the problem statement for now is: devise a signature scheme for signing public keys so that a certificate chain is verifiable in $O(1)$.

I searched Google Scholar and could not find this functionality anywhere in the literature.
The closest notion seems to be from [this paper](https://eprint.iacr.org/2008/037.pdf) where the parties run interactive protocols with each other and some central authority, while I'd rather try to aim for a non-interactive process.
$\newcommand{\KeyGen}{\mathsf{KeyGen}}
\newcommand{\Sign}{\mathsf{Sign}}
\newcommand{\Combine}{\mathsf{Combine}}
\newcommand{\Verify}{\mathsf{Verify}}
\newcommand{\pk}{\mathsf{pk}}
\newcommand{\sk}{\mathsf{sk}}
\newcommand{\Alice}{\mathsf{Alice}}
\newcommand{\Bob}{\mathsf{Bob}}
\newcommand{\Charlie}{\mathsf{Charlie}}$


## Ad-hoc solution idea
As an exercise, I tried coming up with a new scheme. :slightly_smiling_face: 
To start, I find it helps to view the entire certificate chain that authorizes Davy‘s public key as a single object.
Namely, it is a single Alice-signature over Davy‘s key -- it just happened to be iteratively derived via the intermediary signers Bob and Charlie.
Stated like this, we‘re looking for the following kind of interface:
- $\KeyGen() \Rightarrow \sk, \pk$
- $\Sign(\sk_{\Alice}, \pk_{\Bob}) \Rightarrow \sigma_{AB}$
    - $\Sign$ might take additional arguments like the previous signature $\sigma_{XA}$ in an ongoing chain.
- $\Combine(\sigma_{AB}, \sigma_{BC}) \Rightarrow \sigma_{AC}$
- $\Verify(\pk_{\Alice}, \pk_Z, \sigma_{AZ}) \Rightarrow 0/1$
    - Might also take, eg, the previous signature $\sigma_{XA}$ in an ongoing chain.

Security needs to be defined similarly to transitive signatures, with an adversary unable to output a signature $\sigma_{AZ}$ if the *certification graph* contains no path from Alice to Z.
For this sketch of the idea, I will not even worry about the details like how to model the adversary’s oracle access here.

### Construction
How could Bob‘s signature be used to transform Alice‘s signature over Bob‘s pk into a signature over Charlie‘s pk?
To start, I had the vague intuition that Alice‘s signature probably somehow needs to incorporate a shared secret with Bob.
With a scheme based on discrete logarithms, naturally enough, this might be a Diffie-Hellman secret.
Next, for the Combine function to output short signatures, a pairing-based scheme seems appropriate.
After going through a bit of literature, I started thinking that [the way sequential aggregation works for Waters signatures](https://www.iacr.org/archive/eurocrypt2006/40040472/40040472.pdf) looks like it can be made compatible with embedded Diffie-Hellman secrets.
While skimming over such papers, ie, [this one in particular](https://arxiv.org/pdf/0802.1113), I found it especially interesting to see how Waters signatures have previously been modified for *different* extra signing properties.

Let $G, \hat{G}, G_T$ be groups of prime order $p$ with a bilinear pairing $e : G \times \hat{G} \rightarrow G_T$ and generators $g \in G, \hat{g} \in \hat{G}$.
Here is the construction I came up with, in additive notation:
- $\KeyGen()$:
	- Pick random exponents $a, r \in \mathbb{Z}_p^*$
	- $\sk_\Alice = (a, a \cdot g, r)$
	- $\pk_\Alice = (a \cdot \hat{g}, r a \cdot g, r \cdot g)$
- $\Sign(\sk_{\Alice}, \pk_{\Bob}, \sigma_{XA})$:
    - If $\sigma_{XA}$ is empty (there is no ongoing chain), set $\sigma_{XA,2}=g$
	- Pick random exponent $r’$
	- $\sigma_1 = a \cdot \sigma_{XA,2} - r' \cdot \pk_{\Bob, 2}$
	- $\sigma_2 = r' \cdot \pk_{\Bob, 3}$
	- $\sigma_{AB} = (\sigma_1, \sigma_2)$
- $\Combine(\sigma_{AB}, \sigma_{BC})$:
	- $\sigma_1 = \sigma_{AB,1} + \sigma_{BC, 1}$
	- $\sigma_2 = \sigma_{BC, 2}$
	- $\sigma_{AC} = (\sigma_1, \sigma_2)$
- $\Verify(\pk_{\Alice}, \pk_Z, \sigma_{AZ}, \sigma_{XA})$:
    - $e(\sigma_{AZ, 1}, \hat{g}) =^? e(\sigma_{XA,2}, \pk_{\Alice, 1}) - e(\sigma_{AZ,2}, \pk_{\Bob, 1})$

#### Correctness of verification:
Denote Bob's secret exponents as $b$ and $s$.
If $\sigma_{AB}$ comes from $\Sign$, the verification check becomes

$$\begin{align}
e(\sigma_{AB, 1}, \hat{g}) &= e(a \cdot \sigma_{XA,2} - r' \cdot \pk_{\Bob, 2}, \hat{g}) \\
&= e(a \cdot \sigma_{XA,2} - r' s b \cdot g, \hat{g}) \\
&= e(\sigma_{XA,2}, a \cdot \hat{g}) - e(r' s \cdot g, b \cdot \hat{g}) \\
&= e(\sigma_{XA,2}, \pk_{\Alice, 1}) - e(r' \cdot \pk_{\Bob, 3}, \pk_{\Bob, 1}) \\
&= e(\sigma_{XA,2}, \pk_{\Alice, 1}) - e(\sigma_{AB,2}, \pk_{\Bob, 1})
\end{align}$$

#### Correctness of combination:
If $\sigma_{AB}$ comes from $\Sign(\sk_\Alice, \pk_\Bob, \sigma_{XA})$ and $\sigma_{BC}$ comes from $\Sign(\sk_\Bob, \pk_\Charlie, \sigma_{AB})$,
then $\Combine$ outputs $\sigma_{AC} = (\sigma_1, \sigma_2)$ where

$$\begin{align}
\sigma_1 &= \sigma_{AB,1} + \sigma_{BC, 1} \\
&= a \cdot \sigma_{XA,2} - r' \cdot \pk_{\Bob, 2} + b \cdot \sigma_{AB,2} - s' \cdot \pk_{\Charlie, 2} \\
&= a \cdot \sigma_{XA,2} - r' \cdot \pk_{\Bob, 2} + b r' \cdot \pk_{\Bob,3} - s' \cdot \pk_{\Charlie, 2} \\
&= a \cdot \sigma_{XA,2} - r' s b \cdot g + b r' s \cdot g - s' \cdot \pk_{\Charlie, 2} \\
&= a \cdot \sigma_{XA,2} - s' \cdot \pk_{\Charlie, 2} \\
\sigma_2 &= \sigma_{BC, 2} \\
&= s' \cdot \pk_{\Charlie, 3}
\end{align}$$

which is a valid Alice-signature over Charlie’s public key.

#### Security:
TBD but the intention is that the secrecy of $a \cdot \sigma_{XA,2}$ makes it impossible to forge $\sigma_{AB}$ without knowing Bob’s exponent.
Transitively, forging $\sigma_{AZ}$ should be impossible without knowing the secret exponent of someone who is reachable from Alice in the certification graph.
(Or is this obviously insecure because there is some trivial attack...?! Fwiw, I couldn't spot any.)


## Next steps?
To work on this systematically, the first step would be to properly formalize the desired properties of the scheme (with support for groups of signers instead of single signers).
From this we would see if the candidate direct construction -- or some version of it -- is any good.
Besides, there should also be generic constructions, say, from SNARKs/IVC, which should be specified and compared against the direct construction.
Finally, we might build a light client protocol in a similar spirit as [the one with transitive signatures](https://hackmd.io/@VicChaos/HyQJedYhkx).
