---
layout: post
title: Update to Univariate Sumcheck
date: 2025-10-09
---

ZK papers often make a "univariate vs multilinear" distinction that seems to come from the sumcheck protocol, which is naturally defined over multivariates.
As discussed in a [prior post](), I wanted to explore whether there is a comparable protocol for univariates (without FFTs etc.).

However, I've come to think my earlier idea contained a flaw -- the linear-time FFT-free prover probably cannot be realized after all.

A rethink was needed.

This ultimately resulted in a [new version of the work](https://arxiv.org/pdf/2505.00554), that now includes *two* approaches, in which I have much more confidence.

The first one is a way to use the multivariate sumcheck protocol for univariates in Lagrange basis.
Specifically, it is a new "adaptor" to emulate multilinear extensions with univariate extensions (linear-time unlike, eg, the approach from the [Logup-GKR paper](https://eprint.iacr.org/2023/1284.pdf)).

The other is a reduction from univariate sumcheck to Gemini, building on [this post](https://hackmd.io/@relgabizon/ryGTQXWri).

Interestingly, in both protocols, the reduction and the multilinear evaluation perfectly align.
This means they can be early-stopped any time and the remaining claim can be handled with Aurora.
Because Aurora is run on a small instance, the prover is still linear-time overall.
Of course, this isn't a new idea but I find it noteworthy how *easy* and naturally it comes here (other published papers need extra translation steps!).
