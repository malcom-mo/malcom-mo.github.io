---
layout: post
title: Why is there no univariate sumcheck?
date: 02-05-2025
---

Is there a sumcheck PIOP that
- only uses oracles to univariate polynomials (unlike the [standard multivariate sumcheck protocol](https://eprint.iacr.org/2019/317.pdf)),
- has a linear-time prover (unlike [Aurora](https://eprint.iacr.org/2018/828.pdf)’s quotient-based protocol),
- sends no intermediary oracles (unlike the protocol from [this article](https://hackmd.io/@relgabizon/ryGTQXWri))? :thinking:

The last restriction also eliminates the common-sense approach of combining the standard multivariate protocol with an extra multilinear evaluation proof such as [Gemini](https://eprint.iacr.org/2022/420.pdf)’s, [Zeromorph](https://eprint.iacr.org/2023/917.pdf), [Logup-GKR](https://eprint.iacr.org/2023/1284.pdf)’s or [HyperKZG](https://eprint.iacr.org/2024/2099) because all of those send intermediary oracles (to quotients or even and odd parts).

I think the short answer is no.
My longer answer - still just an intuition - is simply that multivariate oracles are like multiple univariate oracles in one, so, of course, you cannot replace the former with the latter without sending further auxiliary oracles.
This feels true regardless of the chosen basis, so little stands to be gained from any potential roots-of-unity tricks when switching to the Lagrange basis.
I’ll expand on the last statement in the remainder of this post because it is somewhat informed by a **fun attempt to construct a 3/3 protocol** using the Lagrange basis, failing at it, and then thinking about where it failed. :smile:
$\newcommand{\uni}{\mathsf{uni}}
\newcommand{\unex}{\mathsf{unex}}
\newcommand{\mlin}{\mathsf{mlin}}
\newcommand{\mlex}{\mathsf{mlex}}$

## Results
I did get the following results that are best described [in the accompanying PDF](https://arxiv.org/pdf/2505.00554) and which I think are kind of cool even if they do not answer the starting question affirmatively:
- a new way of applying the sumcheck protocol to univariate polynomials, that is, an oracle-free linear-prover interactive reduction from univariate sumcheck ($\sum_i g(f_1(w^i), \dots, f_d(w^i))$) to multilinear evaluation ($\mlin(f_1)(r), \dots, \mlin(f_d)(r)$),
- a new linear-prover polylog-verifier *single-round* reduction from multilinear evaluation ($\mlin(f)(r)$) to univariate evaluation ($f(t)$) that uses intermediary oracles.

Here, $\mlin(f_j)$ is the multilinear polynomial with the same monomial coefficients as $f_j$.
This is different from what you would get if you applied the multivariate sumcheck protocol directly to $\sum_i g(f_1(w^i), \dots, f_d(w^i))$.
Then, you’d end up having to evaluate the multilinear extension polynomials of the vectors $(f_j(1), f_j(w), \dots)$ which is not exactly the same as evaluating $\mlin(f_j)$.


## Motivation
While learning about SNARKs, I often came across what turns out to be a somewhat misleading dichotomy, strictly speaking:
namely, there was an impression that SNARKs would have to *either*
- encode the to-be-proven computation using univariate polynomials and then rely on quotienting techniques, *or*
- encode the to-be-proven computation using multilinear polynomials and then use the sumcheck protocol.

In another post, I plan to cover multivariate quotienting. :shushing_face:

In this post, I want to address the question of whether there is a proper univariate analogue of the multivariate sumcheck protocol.
I’m arbitrarily defining “proper” as sending no intermediary oracles (neither quotients nor anything else) and having a linear-time prover implementation like the multivariate sumcheck protocol.


## Initial expectations
The initial idea was based on the good old - emphasis on *old* - intuition that univariate polynomials are  “nicer” than multivariate ones.
After all, they form a principal ideal domain which multivariates do not.
After all, there is a reason why [Kronecker thought in the 1880s (page 11-12)](https://eudml.org/doc/148487) that multivariate problems generally become more tractable if they can be translated to univariate problems.
This is also why we refer to his technique of mapping a multivariate polynomial like $M(x,y)$ to the univariate polynomial $U(x) = \uni(M) = M(x, x^{\deg_x(M)+1})$, which has the same coefficients, as *Kronecker substitution*.
We also know that univariates defined over *roots of unity* should definitely allow for efficient recursive algorithms since this is what lead [Gauss, around 1805,](https://www.cis.rit.edu/class/simg716/Gauss_History_FFT.pdf) to the first description of the FFT.
Finally, univariate polynomials are used in the Reed-Solomon code, which is a great code when the finite field is large enough.

A priori, I'd take this as ample grounds to guess that univariate polynomials should have some kind of analogue of the sumcheck protocol -- because... well, why shouldn’t they? :shrug:


## Notations
Let $H$ be the multiplicative subgroup that consists of the $2^m$th roots of unity $\\{1, w, \dots, w^{2^m-1}\\}$.
If $U$ is a univariate polynomial in $x$, the coefficient before $x^i$ is denoted as $U[i]$.
If $M$ is an $m$-variate polynomial in $x_1, \dots, x_m$, the coefficient before $x_1^{i_1} \cdots x_m^{i_m}$ is denoted as $M[i]$, where $i$ is the integer with digit decomposition $i_1, \dots, i_m$ where $i_1$ is the least significant digit (in [mixed-radix](https://en.wikipedia.org/wiki/Mixed_radix), ie,
$i = \sum_{j \in [m]} i_j \cdot \prod_{k \in [j-1]} (\deg_{x_k}(M)+1)$).
We also define the following polynomials:

| Polynomial | Definition | |
|-|-|-|
|$\uni(v)$         | $\uni(v)(x) = \sum_i v_i x^i$                                             | Univariate polynomial with coefficient vector $v$
|$\uni(M)$        | $\uni(M)(x) = \sum_i M[i] x^i$                                          | Univariate polynomial with the same coefficient vector as the multivariate polynomial $M$ (Kronecker substitution)
|$\unex(v)$      | $\unex(v)(w^i) = v_i$                                                        | Univariate extension polynomial (wrt $H$) of the vector $v$
|$\mlin(U)$      | $\mlin(U)(x_1, \dots, x_m)$ $= \sum_i U[i] x_1^{i_1} \cdots x_m^{i_m}$ | Multilinear polynomial with the same coefficient vector as the univariate polynomial $U$
|$\mlex_D(v)$ | $\mlex_D(v)(i_1, \dots, i_m) = v_i$                                                     | Multilinear extension polynomial (wrt domain $D$) of the vector $v$


## Existing works
Let me highlight three related works as points of comparison for my univariate sumcheck idea.
First, [Bootle et al.’s Eurocrypt 2022 paper Gemini](https://eprint.iacr.org/2022/420.pdf) contains (among other things) a 2-phase IOP where, in the first phase (henceforth, Gemini-1), constraint satisfiability is reduced to multilinear evaluation and then, in the second phase (henceforth, Gemini-2), multilinear evaluation is reduced to univariate evaluation.
Next, there is the combo of Spartan and the HyperKZG commitment scheme described in [Zhao et al.’s MicroNova paper](https://eprint.iacr.org/2024/2099).
Here, constraint satisfiability is also reduced to multilinear evaluation (this is Spartan, abstracting away details) and multilinear evaluation is then also reduced to univariate evaluation (HyperKZG) -- however, the univariate oracle that gets evaluated is different from Gemini, it’s easier to instantiate.
Finally, there is the protocol by [Gabizon et al.](https://hackmd.io/@relgabizon/ryGTQXWri) which reduces univariate polynomial indentitites over roots of unity to univariate evaluation.
This is related because such polynomial identities capture constraint satisfiability when the polynomials interpolate the satisfying assignment vector and, furthermore, checking domain identities is a generalization of univariate sumcheck.

All of the above protocols have a very similar structure.
The difference I want to point out regards the kind of univariate polynomial oracle that gets queried.
Denoting the witness vector (effectively the satisfying assignment for the constraint system) as $v$, the picture looks like this:

|                      | Evaluated multilinear          | Required  oracle |
|-|-|-|
|Gemini-1 + Gemini-2 | $\mlex_{\\{\pm 1\\}^m}(v)$  | $\uni(\mlex_{\\{\pm 1\\}^m}(v))$
|Spartan + HyperKZG | $\mlex_{\\{0,1\\}^m}(v)$      | $\uni(v)$
|Gabizon et al.             | none (explicitly)                  | $\unex(v)$
|My protocols              | $\mlin(\unex(v))$                | $\unex(v)$

I’d argue that the Gabizon et al. protocol implicitly performs a multilinear evaluation of $\mlin(\unex(v))$ just like mine, so the two protcols seem equivalent (mine probably being worse by concrete constants?).
But by making the 2-phase structure explicit, I’d say there is an advantage from being able to easily batch together multiple executions of the second phase.

Next, I’ll briefly go over my protocols and show what I think is interesting about them, namely **how an attempt to “stay univariate” still ended up having to evaluate $\mlin(\unex(v))$**.


## New protocol(s)
My protocols are constructed somewhat differently than prior ones, though they seem to lead to comparable outcomes as stated above.
Specifically, all my protocols are consequences of applying the standard sumcheck protocol to *univariate* polynomials in a particular way.
[The PDF](https://arxiv.org/pdf/2505.00554) explains it better but the high-level flow is as follows.
- We fix a domain $H’$ for which there is a polynomial map $\varphi$ such that the image of $H’$ under $\varphi$ is $H$.
- We then define multivariate polynomials whose sum over $H’$ is the same as our univariates’ sum over $H$ by composing the univariate polynomials with $\varphi$ (this might be called a [pullback](https://en.wikipedia.org/wiki/Pullback_(differential_geometry))).
- Running the multivariate sumcheck protocol on these multivariates leads to round polynomials which just so happen to be small enough to be sent explicitly at the start. However, their degree doubles each round, leaving us with two choices: *either*
    - we send them as oracles, in which case the final claim is a simple univariate evaluation of $g(f_1(\varphi(r)), \dots, f_d(\varphi(r)))$ (but instantiating these oracles seems complicated, especially for a linear-time prover...), *or*
    - we observe that the round polynomials can be equivalently expressed by lower-degree polynomials if we perform a *modulo reduction*, in which case the final claim magically turns into a multilinear evaluation. :astonished:
- Multilinear evaluation is a special case of univariate sumcheck, an inner product, when we carefully construct a certain kind of Lagrange basis polynomial.
- Therefore, we run the same protocol twice overall (like Gemini’s two phases). First, the protocol *without oracles* reduces the univariate sumcheck claim into a multilinear evaluation. Then, the version of the protocol *with oracles* reduces multilinear evaluation to univariate evaluation.

I intentionally kept this overview brief because further details are anyway [in the PDF](https://arxiv.org/pdf/2505.00554).

Before concluding, I just want to drop a few basic facts that make it kind of obvious-in-hindsight why this attempt of “staying univariate” still ended in a multilinear evaluation.
$\newcommand{\low}{\mathsf{low}}
\newcommand{\high}{\mathsf{high}}
\newcommand{\even}{\mathsf{even}}
\newcommand{\odd}{\mathsf{odd}}
\newcommand{\fold}{\mathsf{Fold}}$

## Basic facts
If $f$ is a univariate polynomial of degree at most $2^m-1$, let’s denote the even-odd and low-high decompositions of $f$ as

$$\begin{align*}
f(x) &= f^{\low}(x) + x^{2^{m-1}} f^{\high}(x) \\
        &= f^{\even}(x^2) + x f^{\odd}(x^2).
\end{align*}$$

When keeping track of the coefficients during modulo reduction, one sees that the remainder of $f$ modulo $x^{2^{m-1}}-r$ is

$$
f \% (x^{2^{m-1}} - r) = f^{\low} + r f^{\high}.
$$

Let’s also introduce the following notation for even-odd folding,

$$
\fold[r](f) = f^{\even} + r f^{\odd},
$$

which Gemini, HyperKZG etc. all perform internally.
Next, let’s recall what folding and taking remainders do to evaluations.
First, we know that the even part and odd parts’ values on squares (in particular, on $2^{m-1}$th roots of unity) are

$$\begin{align*}
f^{\even}(w^{2i}) = \frac{f(w^i) + f(-w^i)}{2} \qquad\qquad f^{\odd}(w^{2i}) = \frac{f(w^i) - f(-w^i)}{2 w^i}.
\end{align*}$$

And for modulo reduction, we know that the remainders of $f$ modulo $x^{2^{m-1}} \pm 1$ take the same values as $f$ on the $2^{m-1}$th roots of unity and of their $w$-coset (because those are the respective vanishing sets of $x^{2^{m-1}} \pm 1$).

The similarities of the two ways of folding do not end there.
From keeping track of coefficients, it can be seen that they relate to multilinear reinterpretation as:

$$\begin{align*}
\mlin(f) &= (\mlin(f^{\low}) + x_m \mlin(f^{\high}))(x_1, \dots, x_{m-1}) \\
         &= (\mlin(f^{\even}) + x_1 \mlin(f^{\odd}))(x_2, \dots, x_m).
\end{align*}$$

Finally, when carrying out $m$ steps of folding for any $r_1, \dots, r_m$, this means

$$\begin{align*}
(\cdots (f \% (x^{2^{m-1}}-r_m)\cdots) \% (x-r_1) &= \fold[r_m](\cdots \fold[r_1](f) \cdots) \\
                          &= \mlin(f)(r_1, \dots, r_m).
\end{align*}$$


## Lessons learned
When starting with univariate polynomials, I expected the outcome of a sumcheck-type reduction to be a nice and simple univariate evaluation.
But, (not?) surprisingly, by using the sumcheck protocol, the requirement for multilinear evaluation comes naturally as a consequence of the interactive reduction pattern and nothing else.
That is, in each round the verifier’s randomness must reduce the sumcheck claim to a claim about a smaller domain.
But both straightforward ways of splitting up a univariate polynomial into two smaller parts automatically boils down to re-interpreting the polynomial as a multilinear polynomial, hence why the reduction ends in a multilinear evaluation.

Multivariate evaluation, in turn, is a strictly more powerful operation than univariate evaluation.
This can be seen from different angles, for example:
- Multivariate evaluation corresponds to performing multiple independent modulo reductions in sequence whereas univariate evaluation is a single one.
- Multivariate oracles can be used as univariate oracles but not vice versa.

This way, a multivariate oracle is like many univariate oracles in one.

So, at least in my mind, this is why it makes sense that there is probably no way to get rid of the intermediary oracles in sumcheck-like reduction protocols. :smiley:
