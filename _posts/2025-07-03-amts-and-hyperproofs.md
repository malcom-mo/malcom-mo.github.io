---
layout: post
title: Notes on AMTs and Hyperproofs
date: 2025-07-03
---

[AMTs](https://dspace.mit.edu/bitstream/handle/1721.1/129845/scalable_thresh.pdf) and [Hyperproofs](https://eprint.iacr.org/2021/599.pdf) are vector commitment schemes built from [KZG](https://www.iacr.org/archive/asiacrypt2010/6477178/6477178.pdf) and, respectively, [PST](https://eprint.iacr.org/2011/587.pdf) polynomial commitments.
Both share the properties that proofs for a size-$2^m$ vector are of size $m$ and that the opening proofs for all elements can be organized in a binary tree so that an update to a single vector entry requires updating the proof tree in only $m$ places.
Unlike [another KZG-based vector commitment](https://eprint.iacr.org/2020/527.pdf) (where proofs have size 1, are natively batchable but not efficiently updatable), neither AMTs nor Hyperproofs have native multi-point openings.
The proof size and the lack of multi-point openings seem to be the only drawbacks of AMTs and Hyperproofs compared to KZG.

Multi-point openings for PST are given by the [Boomy](https://eprint.iacr.org/2023/1599.pdf) paper.
[See this previous post for an introduction to Boomy]({% link _posts/2025-06-11-boomy.md %}).
In principle, this yields batch openings for Hyperproofs, too.
But does Boomy batching benefit from the proof tree so that these batch openings are also *efficient*?

In this post, I wanna try to show:
- A sense in which AMT proofs and Hyperproofs are “really the same”.
- That AMT proofs cannot be naturally compressed down to KZG proofs.
- That multiple Hyperproofs cannot be naturally compressed down to a batched (Boomy) proof.

*Note: I’m using multi-point openings and batching synonymously.*
$\newcommand{\uni}{\mathsf{uni}}$

## AMT Proofs $\approx$ Hyperproofs :face_with_peeking_eye:
I believe it’s fair to say that: *when using roots of unity*, the AMT scheme is Hyperproofs under Kronecker substitution.
Let me explain.
By Kronecker substitution I mean mapping a multivariate $f$ to a univariate $\uni(f)$ with the same coefficient vector.
In case $f$ is $m$-linear, this comes from replacing each variable as
\begin{align}\tag{1}
x_i \mapsto x^{2^{i-1}}
\end{align}
or $\uni(f) = f(x, x^2, \dots, x^{2^m-1})$.
For multiplication, care has to be taken about the degrees.
Eg, if $f$ and $g$ are both multilinear, the product has degree 2 in each variable.
Then, we must define $\uni$ to set the degrees as $x_i \mapsto x^{3^{i-1}}$ in order to get $\uni(fg) = \uni(f) \uni(g)$.
Generally, for $\uni(fg) = \uni(f) \uni(g)$ to hold, $\uni$ must set the degrees at least as
\begin{equation}\tag{2}
x_i \mapsto x^{(\deg_1(f) + \deg_1(g) +1) \cdots (\deg_{i-1}(f) + \deg_{i-1}(g) + 1)}.
\end{equation}

For the AMT-Hyperproof equivalence to work, I am assuming that we use Hyperproofs with roots of untiy as well.
Specifically, instead of interpolating over $\\{0,1\\}^m$, I am thinking of a variant of Hyperproofs where our multilinear polynomial $f$ is interpolated over the set $D = \\{(w^i, w^{2i}, \dots, w^{2^{m-1} i}) \mid i \in [0, 2^m-1]\\}.$
Here, $w$ is the primitive $2^m$th root of unity.
For points of this form, the PST/Hyperproofs opening identity is
$$\begin{align}\tag{3}
f - f(w^i, \dots, w^{2^{m-1} i}) = \sum_{j \in [m]} q_j \cdot (x_j - w^{2^{j-1} i}).
\end{align}$$
By inspecting the degrees, $q_j$ cannot depend on $x_j$.
In fact, if the $q_j$ are computed iteratively, starting with $q_m$, then $q_j$ is linear in $x_1, \dots, x_{j-1}$ and does not depend on $x_j, \dots, x_m$.

Now, let’s look at AMT proofs when interpolating the same vector as in Hyperproofs with a univariate polynomial $u$ over the roots of unity $\\{w^i \mid i \in [0,2^m-1]\\}$.
The crucial observation is that the multilinear $f$ from Hyperproofs and the univariate $u$ have the same coefficient vector, $u = \uni(f)$.
Indeed, if $u = \sum_{k \in [0,2^m-1]} c_k x^k$, then the bit decomposition of $k$ gives

$$\begin{align*}
u(w^i) &= \sum_{k \in [0,2^m-1]} c_k w^{i k} \\
&=  \sum_{k \in [0,2^m-1]} c_k w^{i (k_1 + \cdots + 2^{m-1} k_m)} \\
&=  \sum_{k \in [0,2^m-1]} c_k w^{i k_1} \cdots w^{2^{m-1} i k_m}
\end{align*}$$

which is the multilinear polynomial with the same coefficients evaluated at $(w^i, \dots, w^{2^{m-1} i})$.
But because the evaluation matches $u(w^i)$ (for all $i$), that multilinear must be $f$.
Further, the AMT opening identity is

$$\begin{align}
u - u(w^i) = \sum_{j \in [m]} q’_j \cdot (x^{2^{j-1}} - w^{2^{j-1} i}).
\end{align}$$

**This is the same as Equation (3) under Kronecker substitution (1)** because the map (1) fulfils the degree condition from (2) for each multiplication $q_j \cdot (x_j - w^{2^{j-1} i})$.

[My other post described that]({% link _posts/2024-11-11-zeromorph.md %}) the idea of committing to an $m$-variate polynomial $p$ with a univariate powers-of-tau string, under Kronecker substitution, is something like a simplified "mini" version of [Zeromorph](https://eprint.iacr.org/2023/917.pdf).
That is, it is a valid multivariate polynomial commitment scheme.
The post derives, from properties of Euclidean division, that the opening identity for $p(v_1, \dots, v_m) = y$ is built out of a sequence of modulo reductions by $x^{c_j} - v_j$ where $c_j$ depends on the individual degrees of $p$.
For $p$ multilinear, $c_j = 2^{j-1}$.
What does all this mean?
If $(v_1, \dots, v_m)$ is from the domain $D$, we have $v_j = w^{2^{j-1} i}$.
Therefore, the AMT opening identity is a special case of the mini-Zeromorph opening identity. :bulb: 
This means that **AMT and Hyperproofs actually commit to the same *multilinear* polynomial $f$**.
AMT uses mini-Zeromorph (which internally does the Kronecker substitution to map $f$ to the univariate $u=\uni(f)$) and Hyperproofs uses PST.
This is how AMTs and Hyperproofs are essentially “the same thing”.


## Compressing AMT proofs :thinking:
A KZG quotient $q$ satisfies $u-u(w^i) = q \cdot (x-w^i)$, showing that $q$ has degree at most $2^m-2$.

From the fact that the AMT quotients $q_1', \dots, q_m'$ can be obtained by a sequence of Euclidean divisions with remainder, we know that they have degrees $0, 1, 3, \dots, 2^{m-1}-1$.
Eg, $u$ is first divided by $x^{2^{m-1}}-w^{2^{m-1} i}$, giving a quotient and a remainder,
$u = r_m + q_m' \cdot (x^{2^{m-1}}-w^{2^{m-1} i}).$
Since $r_m$ is the remainder modulo $x^{2^{m-1}}-w^{2^{m-1} i}$, it has degree $\leq 2^{m-1}-1$ just like $q_m'$.
$q_{m-1}'$ is obtained by dividing $r_m$ by $x^{2^{m-2}} - w^{2^{m-2} i}$ etc.
Alternatively, we could deduce the degrees from the fact that they are obtained by Kronecker substitution of Hyperproofs quotients.
To explicitly relate AMT and KZG quotients:

$$\begin{align*}
q &= \frac{u-u(w^i)}{x-w^i} \\
&= \sum_{j \in [m]} q_j' \frac{x^{2^{j-1}}-w^{2^{j-1} i}}{x-w^i}.
\end{align*}$$

So, in order to compute a commitment $[q(\tau)]$ from the elements from the AMT proof tree $[q_1(\tau), \dots, q_m(\tau)]$, you would need to perform multiplications by $\tau$ ”in the exponent“ which contradicts computational Diffie-Hellman.


## Aggregating Hyperproofs :thinking:
As Hyperproofs uses PST and multi-point openings for PST are given by Boomy, one might wonder whether multiple Hyperproofs can be compressed down to Boomy proofs.
[For an introduction to Boomy, also see my previous post]({% link _posts/2025-06-11-boomy.md %}).
In one sentence, a Boomy proof for $k$ points $V = \\{v_1, \dots, v_k\\} \subset \\{0,1\\}^m$ consists of commitments to quotients that prove the divisibility of our committed polynomial by the reduced Gröbner basis of the ideal of $V$.
However, Gröbner bases for size-$k$ subsets of $\\{0,1\\}^m$ can get somewhat complicated.
For example, the following Sage code outputs a Gröbner basis of 10 elements for 5 points in $\\{0,1\\}^4$:
```python
from functools import reduce

F = GF(31)
R.<x1,x2,x3,x4> = PolynomialRing(F, order='lex')
V = [(0, 0, 0, 0), (1, 0, 0, 0), (0, 1, 0, 0), (0, 0, 1, 0), (0, 0, 0, 1)]
point_ideals = [R.ideal([x1 - a, x2 - b, x3 - c, x4 - d]) for (a, b, c, d) in V]
ideal_of_V = reduce(lambda I, J: I * J, point_ideals)
G = ideal_of_V.groebner_basis()

print(G)
# [x1^2 - x1, x1*x2, x1*x3, x1*x4, x2^2 - x2, x2*x3, x2*x4, x3^2 - x3, x3*x4, x4^2 - x4]

```
*Note: The code had to describe $I(V)$, which is normally thought of as the intersection of the point ideals.
But because the point ideals are maximal, their intersection is the same as their product.*

To avoid such big Gröbner bases, it turns out we can just change our interpolation set from $\\{0,1\\}^m$ to another.
For example, let’s go back to the Hyperproofs variant that produces AMT proofs under Kronecker substitution and interpolate over

$$D = \left\{\left(w^i, w^{2i}, \dots, w^{2^{m-1} i}\right)\right\}.$$

What is so special about $D$?
It turns out that the fact that all points in $D$ differ in the first coordinate forces a simple structure for the reduced Gröbner basis of the ideal of any subset $V \subseteq D$.
Namely, let $g_1(x_1)$ vanish on the first coordinates, $g_1 = \prod_{v \in V} (x_1-v_1)$, and let $g_i(x_1)$ interpolate the $i$th coordinates over the first coordinates, $g_i(v_1) = v_i$ for all $v \in V, i \in [2,m]$.
Then, the reduced Gröbner basis of the ideal of $V$ is $g_1, x_2-g_2, \dots, x_m-g_m$.
This is interesting because there are only $m$ elements, they can be easily computed and iterative division eliminates one variable after another just like with standard PST quotients.
In the literature, ideals of finite sets of points that all differ in one position and therefore admit such Gröbner bases are said to be in [“shape position”](https://dl.acm.org/doi/pdf/10.1145/190347.190382).
This case is also discussed in the Boomy paper (as one of two classes of nice Gröbner bases - see the previous post for the other one).
The $m=2$ version is also Exercise 14 from Chapter 2.7 of the “Ideals, Varieties and Algorithms” textbook.

### Incompressibility yet again 
For the multi-point opening, let $V \subseteq D$ and let the univariate $f_V(x_1)$ interpolate the values $f(v)$ over the first coordinates, $f_V(v_1) =f(v)$ for all $v \in V$.
The Boomy proof then consists of quotients showing the divisibility of $f-f_V$ by the shape-position Gröbner basis.
We can now make a similar argument as we did for AMT-to-KZG compression.
Namely, that the Hyperproofs tree does not help to produce Boomy proofs efficiently because of too high degrees.

Let's start by computing the expressions of the desired multi-point quotients $Q_1, \dots, Q_m$.
We start from $f = \sum_{i_1, \dots, i_m \in \\{0,1\\}} c_{i_1\cdots i_m} x_1^{i_1} \cdots x_m^{i_m}$ and use the fact that the quotients can be obtained by iterated Euclidean division.
Starting with $Q_m$, we write $f = r_m + Q_m \cdot (x_m-g_m)$.
Since $r_m$ here is the remainder of $f$ when dividing by $x_m-g_m$, we have $r_m = f(x_1, \dots, x_{m-1}, g_m)$.
Then,

$$\begin{align}
f-r_m &= \sum_{i_1, \dots, i_m \in \\{0,1\\}} c_{i_1 \cdots i_m} x_1^{i_1} \cdots x_{m-1}^{i_{m-1}} (x_m^{i_m} - g_m^{i_m}) \\
&= \sum_{i_1, \dots, i_{m-1} \in \\{0,1\\}} c_{i_1 \cdots i_{m-1} 1} x_1^{i_1} \cdots x_{m-1}^{i_{m-1}} (x_m - g_m)
\end{align}$$

and

$$Q_m = \frac{f-r_m}{x_m-g_m} = \sum_{i_1, \dots, i_{m-1} \in \\{0,1\\}} c_{i_1 \cdots i_{m-1} 1} x_1^{i_1} \cdots x_{m-1}^{i_{m-1}}.$$

We continue and divide $r_m$ by $x_{m-1}-g_{m-1}$, getting remainder $r_{m-1} = f(x_1, \dots, x_{m-2}, g_{m-1}, g_m)$ for which it holds that

$$\begin{align}
r_m-r_{m-1} &= \sum_{i_1, \dots, i_{m} \in \\{0,1\\}} c_{i_1 \cdots i_{m}} x_1^{i_1} \cdots x_{m-2}^{i_{m-2}} (x_{m-1}^{i_{m-1}}-g_{m-1}^{i_{m-1}}) g_m^{i_m} \\
&= \sum_{i_1, \dots, i_{m-2}, i_m \in \\{0,1\\}} c_{i_1 \cdots i_{m-2} 1 i_m} x_1^{i_1} \cdots x_{m-2}^{i_{m-2}} (x_{m-1}-g_{m-1}) g_m^{i_m}.
\end{align}$$

Similarly to the case above, this shows that

$$Q_{m-1} = \sum_{i_1, \dots, i_{m-2}, i_m \in \\{0,1\\}} c_{i_1 \cdots i_{m-2} 1 i_m} x_1^{i_1} \cdots x_{m-2}^{i_{m-2}} g_m^{i_m}.$$

Continuing these calculations, for $j \in [2,m]$ we get

$$Q_j = \sum_{i_1, \dots, i_{j-1}, i_{j+1}, \dots, i_m \in \\{0,1\\}} c_{i_1 \cdots i_{j-1} 1 i_{j+1} \cdots i_m} x_1^{i_1} \cdots x_{j-1}^{i_{j-1}} g_{j+1}^{i_{j+1}} \cdots g_m^{i_m}.$$

Finally, the univariate $r_2 = f(x_1, g_2, \dots, g_m)$ gets divided by $g_1$.
The resulting remainder is the unique univariate polynomial of degree less than $\deg(g_1)$ that agrees with $r_2$ at the points where $g_1$ vanishes, that is, on the first coordinates $v_1$ of all $v \in V$.
By construction, that's $f_V.$
From inspecting the equation $r_2-f_V=Q_1 g_1$, the quotient $Q_1$ is univariate in $x_1$ and has degree $\leq \deg(r_2)-\deg(g_1)$ where $\deg(r_2) \leq (m-1)(|V|-1)+1$ and $\deg(g_1) = |V|$.
Hence, $\deg(Q_1) \leq (m-2)(|V|-1)$.

Note that the existing Hyperproofs proof tree elements are the same as the $Q_j$ except that, in the opening proof for $v \in D$, each of $g_2, \dots, g_m$ above is replaced with the respective constant $v_j$ and $g_1$ is $x_1-v_1$.

The point is that the $x_1$-degree of the $Q_j$ keeps increasing, making it impossible to efficiently compute commitments to $Q_j$ given only the proof tree elements.
It would require multiplications by the secret trapdoor elements, again contradicting CDH.


## Follow-up questions :smiley:
- What if we allow additional data on the tree, eg, going from single trees to a kind of forest?
  Might that yield efficient AMT-to-KZG compression or Hyperproofs-to-Boomy aggregation after all, even if at an increased cost to proof updates?
- Can we Kronecker-substitute the Hyperproofs/Boomy batching indentity to get aggregation for AMTs/mini-Zeromorph?
- Otherwise, if we accept that natural compression and aggregation are not possible, is there at least a chance for efficient proof-of-knowledge protocols?
  Eg, could techniques similar to the [KZG multi-commitment multi-point batching protocol](https://eprint.iacr.org/2020/081.pdf) be used? Or some specialized powers-of-tau setup similar to [Groth16](https://eprint.iacr.org/2016/260.pdf)?

