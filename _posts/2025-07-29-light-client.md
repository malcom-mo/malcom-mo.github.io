---
layout: post
title: Light Clients from Transitive Signatures
date: 2025-07-29
order: 2
---

This post is about [our 2024 paper](https://arxiv.org/abs/2410.03347)'s light client design which has optimistic-case $O(1)$ syncing complexity due to the novel use of transitive signatures under the hood.
I'll specify some aspects that were left out of the paper's abstract description.
Then, we will see how the design can be implemented for a particular permissioned BFT protocol, namely Istanbul BFT as used by Quorum.

{: .success}
> [Another post]({% link _posts/2025-07-29-transitive-signatures.md %}) goes deeper on transitive signatures, the underlying cryptographic primitive.

## Main idea
We are in the setting of BFT committee-based blockchains where the validator committee may change after every epoch.
We'll say that $V^{(E)}$ is the set of validators for epoch $E$.
We are investigating how the BFT protocol can be tweaked in order to support light clients which can efficiently (re-)sync with the blockchain after an offline phase.

The standard (old) syncing flow, repeated for all consecutive epochs, works as follows.
- At some point during epoch $E$, the validators of epoch $E+1$ are decided.
- At the end of epoch $E$, the last block in the epoch contains the list $V^{(E+1)}$.
- The commit certificate of this block, therefore, also acts as a signature over $V^{(E+1)}$. 
- A client that is sync'ed up to epoch $E$ and stores $V^{(E)}$ uses the commit certificate to verify $V^{(E+1)}$ and, subsequently, uses $V^{(E+1)}$ to verify all blocks in epoch $E+1$.

Note that validators who leave the committee are assumed to securely erase their secret keys.
If they didn't, attacks would be possible.
But even with this assumption, if the client is out of sync by $m$ epochs, the standard sync flow still requires $O(m)$ work.
Our protocol, by contrast, aims for $O(1)$.

### New protocol for "oligopolies"
Many practical systems allow for an important simplifying assumption: validators often remain in place across epochs.
This does not apply to, say, Ethereum where the signing committee changes each slot.
But it does apply many others.
So, if this is the case, isn't there a chance to drastically simplify light client protocols?

Our new protocol is designed to let the just-returning light client re-use the keys of the last known validators as much as possible.
To this end, each epoch change will come with a new extra signature from which the verifier learns the following: *may the same keys that were OK for verifying epoch $E$ be re-used for $E+1$, too?*
Two design choices make this efficient:
1. Relying on pairing-based signatures allows representing the set of keys "that were OK for verifying epoch $E$" as a single aggregated public key (APK), $k^{(E)}$.
2. [Transitive signatures]({% link _posts/2025-07-29-transitive-signatures.md %}) allow compressing consecutive signatures into a single one that asserts: *may the same keys that were OK for verifying the last epoch $E$ be re-used for $E+m$, too?*

### Details
Our new protocol determines a special subset of validators according to a deterministic rule so that all validators know it.
We think of it as the *longest-running quorum*.
The protocol requires that the honest validators output a second signature at the end of each epoch.
This signature is computed in one of two ways, depending on whether the same longest-running quorum still makes up a quorum for the next round.
If so, the signed value is just the epoch counter.
Otherwise, eg, if one of the validators from the quorum leaves, it is the APK $k^{(E)}$ of the new longest-running quorum.

For simplicity, we say that "$k^{(E)}$ gets signed", understanding that some of these signatures are computed based on the epoch counter alone.

We always think of $k^{(E)}$ as an APK that should be used (by syncing clients in the future) to **verify** the $E$th epoch.
This means that $k^{(E)}$ is the APK of a subset of $V^{(E-1)}$, the validators of epoch $E-1$.
It gets decided and, potentially, signed at the **end** of epoch $E$.
This is because it can only be signed once it is known whether the same validators A) actually signed off on the $E$th epoch and B) will stay on for the next epoch, in particular, whether $k^{(E+1)}=k^{(E)}$.

{: .info}
> This is a "liveness" concern that the paper's model had ignored for simplicity.

### Example flow
Consider the good case in which the longest-running quorum remained static over a run fo $m$ epochs.
In particular, suppose a light client is sync'ed up to epoch $E$, meaning that it stores the list $V^{(E)}$, but the current epoch is already $E+m$.
When the light client sends a sync request to a full node, the full node will send a bit vector specifying the APK $k^{(E+1)}$ and an aggregate of the transitive signatures over $k^{(E+1)}, \dots, k^{(E+m-1)}$ to the client.
The aggregated signature will allow the light client to confirm that, indeed, the same quorum that was a quorum in epoch $E$ (and thus authorized to sign the hand-off to epoch $E+1$) is also authorized to sign the hand-off to epoch $E+m-1$.
Effectively, the client may now safely "jump ahead" and verify $V^{(E+m-1)}$ against $V^{(E)}$.
Given $V^{(E+m-1)}$, the light client can use the standard flow for the last mile, that is, to verify $V^{(E+m)}$ and then any block from the current epoch $E+m$.

The table shows what gets decided and signed when:

| Epoch | Current validators | Next validators | Gets signed at end of epoch |
| - | - | - | - |
| $E-1$ | $V^{(E-1)}$ | $V^{(E)}$ | $V^{(E)}, k^{(E-1)}$ |
| $E$ | $V^{(E)}$ | $V^{(E+1)}$ | $V^{(E+1)}, k^{(E)}$ |
| $E+1$ | $V^{(E+1)}$ | $V^{(E+2)}$ | $V^{(E+2)}, k^{(E+1)}$ |
| $\dots$ | | | |
| $E+m-1$ | $V^{(E+m-1)}$ | $V^{(E+m)}$ | $V^{(E+m)}, k^{(E+m-1)}$ |
| $E+m$ (current) | $V^{(E+m)}$ | (not yet known) | (not yet known) |


## Better tolerating asynchrony
For the protocol to be efficient, we want long runs where the same $k^{(E)}$ gets signed after each epoch.
But, unfortunately, even if the same quorum validators remain active, network asynchrony can still degrade the protocol.
This is because a validator will only sign on the same $k^{(E)}$ when it has *locally seen* the quorum validators' signatures (over the prior round).
In an asynchronous network, messages can always be delayed or out of order.
Therefore, it is possible that the longest-running quorum gets prematurely replaced because those validators' signatures arrive late even if they do eventually arrive.

Wouldn't it be nice if the future light client's efficiency did not depend so much on day-to-day network fluctuations?
There is a straightforward way to address this concern.
Namely, we stagger the protocol by $d$ rounds:
- $k^{(E)}$ does not get defined as an APK of validators from $V^{(E-1)}$ who signed $V^{(E)}$ at the end of epoch $E-1$.
- Instead, it will be the APK of validators from $V^{(E-d-1)}$ who signed $V^{(E-d)}$ at the end of epoch $E-d-1$.
- Depending on the epoch length, it is overwhelmingly likely that all required signatures will have arrived at all validators by the end of epoch $E$.
- This way, the protocol only helps the light client skip ahead to $V^{(E+m-d-1)}$ and the "last mile" of standard syncing is a bit longer.



## Istanbul BFT integration
In the [Quorum](https://github.com/Consensys/quorum/) blockchain's [Istanbul BFT](https://arxiv.org/pdf/2002.03613) consensus, each block goes through the following rounds:
- Pre-prepare. The proposer/leader broadcasts the block to all validators.
- Prepare. The validators respond by broadcasting prepare-votes/signatures over the proposal.
- Commit. After seeing enough prepare votes, validators broadcast commit-votes over the proposal.
- After seeing enough commit-votes, everyone locally decides the block.
  Formally, each validator forms a so-called commit certificate which is just the collection of commit-votes (of which there are, by definition, at least $2f+1$).

A pre-prepare will only be answered if the "previous block" referenced in the proposed block already has a commit certificate.

### Adding the new signatures
We integrate our new signatures over $k$ into this structure.
First, we want every commit certificate to also contain the validators' signatures over $k$.
For validators to sign $k$ (and the same $k$!), a unique description of $k$ must be included in the pre-prepare/proposal.
Then, the consensus logic will ensure that all commit certificates authenticate the same $k$.

Concretely, we must add two rules:
1. A pre-prepare is only recognized as valid if it includes the description of an APK $k$ that is the APK of a quorum of validators who all sent commit-votes over the *previous* end-of-epoch block (or, for the staggered variant, $d$ epoch blocks before).
2. A commit-vote is only recognized as valid if it comes with a separate signature, made with our special scheme, over $k$ from the proposal.

For the description of $k$ we can choose a bit vector of the validators whose keys go into $k$.
To check rule 2 using the bit vector, we just check if we indeed received the required commit-votes from the indicated validators (and that there are $2f+1$ of them).
Note that if we put $k$ itself in the proposal and not a bit vector representation, then checking rule 2 would be inefficient: the validators would have to see if there is *any* subset of validators who issued commit-votes and whose keys added up to $k$.

It is the proposer's job to choose $k$.
For consensus not to slow down, the proposer may always track how long each validator is active and always choose the $2f+1$ longest-running validators.
Ideally, modulo momentary asynchrony as mentioned above, this should result in long runs of the same $k$ which makes future light clients $O(1)$ as desired. :raised_hands:


## Final notes
### Implementation
An implementation of the signature scheme and a light client *simulation* with mock data is included in the [code released along with our paper](https://github.com/malcom-mo/acsac24).
<!--An exmaple integration of our protocol into the Quorum blockchain is available [here](asdf).-->


### "Prover cost"
One of the reasons we focus on the Quorum blockchain is that it uses Istanbul BFT, the same consensus that was used by Celo1.
We can draw a comparison between our protocol and [Celo1's light client](https://eprint.iacr.org/2021/1361.pdf).
This is interesting because back then, Celo1 ran one of the historically largest zero-knowledge proofs for their light clients.
The circuit was of size $2^{28}$ and the proving cost was, accordingly, quite high.
Compare this with our signature-based solution:

{: .success}
> In our protocol, prover cost is merely signature aggregation.
