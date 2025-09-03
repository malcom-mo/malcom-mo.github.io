---
layout: post
title: Notes on Zeromorph
date: 11-11-2024
---

Here is a cute simple explanation of the underlying idea (roughly, I think) of how [Zeromorph](https://eprint.iacr.org/2023/917.pdf) turns a univariate polynomial evaluation argument into an $m$-variate evaluation argument using $m$ quotients.
The paper focuses on the $m$-linear case, so we are even slightly generalizing here.
<!--No commutative diagrams or fancy terminology such as "Kronecker substitution" or "module isomorphism though not a ring isomorphism" will appear.-->

## Preliminaries
- **Notation:** If $p$ is a univariate polynomial in $x$ over a field $F$, its coefficient in front of $x^j$ is indexed as $p[j]$ and $\deg(p)$ denotes its degree.
- Given univariate polynomials $p$, $m$ and $r$, $r$ is the remainder of $p$ modulo $m$ if and only if there exists a (quotient) polynomial $q$ such that $$p = r + q \cdot m$$ and $\deg(r)<\deg(m), \deg(q) \leq \deg(p) - \deg(m)$.
- By the *factor theorem*, a univariate polynomial $p$ vanishes at $t \in F$ if and only if $p$ is divisible by $x-t$.
- A generalization is the *remainder theorem* by which, for any $t$, $p$ can be uniquely decomposed as $p = p(t) + q \cdot (x-t)$.
  That is, the value (constant polynomial) $p(t)$ is the remainder of $p$ modulo $x-t$.


## Higher-degree factors
The following is a generalized remainder theorem which can be proven from the basic rules for substituting variables when performing modulo reduction algorithmically.

**Proposition ([Theorem 2.12 from here](https://www.tandfonline.com/doi/abs/10.1080/0020739X.2018.1522676)):**
For any $t \in F$, any univariate polynomial $p$ of degree $d$ can be uniquely decomposed as
$$p = r + q \cdot (x^k-t)$$
such that $\deg(q) \leq d-k$, $\deg(r) < k$ and the coefficients of $r$ are
$$r[i] = \sum_{j \in [0, \left\lfloor{\frac{d-i}{k}}\right\rfloor]} p[i+jk] \cdot t^j.$$

**Proof:**
Recall the substitution method for computing remainders modulo $x^k-s(x)$ for any $s$ with $\deg(s)<k$:
- While $\deg(p) \geq k$:
    - In the expression $p(x) = \sum_{i \in [0,d]} p[i] \cdot x^i$, replace every occurence of $x^k$ with $s(x)$.
    - Set $p$ equal to the resulting expression.

If $s$ is a constant polynomial equal to $t$, the algorithm takes only one iteration.
After this iteration, all terms of the form $p[jk+i] \cdot x^{jk+i}$, for $i<k$, have been replaced with $p[jk+i] \cdot t^j \cdot x^{i}$.
Thus, all $k$th coefficients of $p$, starting from the $i$th, "land" before $x^i$ in the remainder, scaled by powers of $t$.
**QED**

This can be interpreted as splitting up $p$ into $k$ "subpolynomials" where each one only has every $k$th coefficient of $p$ (the first one starting with $p[0]$, the second with $p[1]$ etc.).
The coefficients of the remainder of $p$ modulo $x^k-t$ are then the evaluations of the subpolynomials at $t$.


## Multivariate evaluation
Taking the above theorem, what if we evaluate $r$ at, say, $u$?
The result will be a combination of coefficients of $r$ with powers of $u$.
Overall, we will have combined coefficients of $p$ with terms of the form $t^i \cdot u^j$.
That is, we will have successfully interpreted $p$ as a bivariate polynomial and will have evaluated that bivariate at $(t,u)$.
Therefore, after having authenticated (a commitment to) $r$ with (a commitment to) the quotient modulo $x^k-t$, the evaluation of $r$ can be proven using (a commitment to) a standard quotient modulo $x-u$.
This way, we will have proven the evaluation of the bivariate.

To be clear, the bivariate is the unique bivariate polynomial whose coefficients are the same as $p$'s coefficients.

Alternatively, instead of evaluating $r$ at $u$, we could also repeat the above theorem and perform a modulo reduction by $x^{k'}-u$ for $k'>1$.
We'd get another remainder $r'$, we could evaluate that remainder (and prove said evaluation with a standard quotient) and we would then have interpreted $p$ as a 3-variate.
The pattern is clear: we can iterate this process as long as we want -- or as long as the remainder's degree is more than 1.

And this is what I'd already call a baby version of Zeromorph, maybe always using degrees $k$ that are powers of 2 in order to interpret $p$ as multilinear!


## To Zeromorph
The setting in Zeromorph is: we have the *values* of a multilinear polynomial $f$ on some domain and we would like to commit to $f$ and prove its evaluations outside the domain using a univariate commitment scheme.
Based on the above, one possible approach is to first compute the *coefficients* of $f$ and then commit to the univariate polynomial $p$ which has the same coefficients.
Then, evaluations of $f$ can be proven using univariate quotients as discussed (ie, the re-interpretation of $p$ as a multivariate will lead again to $f$).

However, this is not what Zeromorph does.
The computation of coefficients from evaluations -- an $m$-dimensional discrete Fourier transform (DFT) -- is expensive and can be avoided.
Instead, the paper defines a "univariatization map" that transparently takes care of the values-to-coefficients transformation and that preserves the existence of univariate quotients similar to the ones described here.


### Future work
Let me sketch a potential alternative construction, specific to KZG.
Namely, I'd imagine that a suitable preprocessed SRS gives another simple way to avoid the expensive $m$-variate DFT computation.
Then, the above logic might apply directly.
Suppose that we are in the Zeromorph setting and $f$ is $m$-variate and has degree $d$ in each variable.
We are given its evaluations over $H^{m}$ where $H \subset F$ has size $d+1$.
We know that there exist $m$-variate Lagrange basis polynomials $\ell_{h}, h \in H^m$ such that $f = \sum_{h \in H^m} f(h) \cdot \ell_h.$
Now, the SRS should contain KZG commitments to the univariate polynomials $b_h$ that have the same coefficients as $\ell_h$.
Then, by linearity, the univariate polynomial $p$ with the same coefficients as $f$ can be constructed from the given evaluations of $f$ as $p = \sum_{h \in H^m} f(h) \cdot b_h$.
This way, we can commit to $p$ without an expensive $m$-dimensional DFT.
(I haven't worked out how the commitments to the quotients would be computed.)
