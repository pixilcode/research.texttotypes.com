+++
title = "Negation as Failure"
description = "An overview of the concept of 'Negation as Failure'."
date = 2024-09-30

[taxonomies]
tags = ["overview", "fixed-point parsing"]
+++

# Negation as Failure

In logic, there are things that are true and things that are not true. Given
some proposition $P$, if $P$ is true, it is represented simply as $P$. If $P$ is
false, it is written as $\neg P$.


## Standard Logic Systems

In standard logic systems, something is false only if a proof can be supplied of
its false nature. For example, given a set $S = \{2, 4, 6, 9\}$ and a proposition
$P = \forall x \in S, \text{isEven} (x)$, we can prove $\neg P$ by supplying the proof
$\neg \text{isEven}(9)$.

However, given a set $T = \{2, 4, 6, n\}$ where $n$ is not known and a
proposition $Q = \forall x \in T, \text{isEven}(x)$, we cannot supply a proof
$n is even$ or a proof $\neg \text{isEven}(n)$. Thus, we cannot prove either $P$ or
$\neg P$.


## Negation as Failure

In a system like Datalog, negation is treated slightly differently. Rather than
requiring a proof that a proposition is false, it is sufficient to show that the
proposition isn't true based off of the information that we have. This form of
negation is sometimes differentiated as $\text{not} P$, which is what I will use.

Returning to our previous example where $T = {2, 4, 6, n}$ and $Q = \forall x
\in T, \text{isEven}(x)$, it still isn't possible to prove $\neg Q$.  However,
we are able to prove $\text{not} Q$. This is because no proof exists for $Q$, which,
using negation as failure, implies $\text{not} Q$.


## Usefulness

Why is there a difference in treatment? It all comes down to the goals and
capabilities of each system. In math, the goal is to have an exact truth. There
should be no "holes" or inconsistencies in the logic. You are able to manipulate
and many different theoretical concepts. In this case, it is very helpful to
treat something as false only if there is a proof that it is false.

In computation, on the other hand, you are much more limited in what you have
access to. Every piece of data must be encoded in the system. This means that
you may not have access to all the facts that you need.

In addition, your goal is to produce *some* result that you can act on. While it
can be useful in some situations, "I don't know" often means that you can't
continue with the computation. So, instead of requiring a proof of false-ness
(which may not exist), it is helpful to ask whether a proof of truthfulness
exists in the current system, since that will always be either true or false.

Thus, "negation as failure" is used in computation because of the incompleteness
of data and facts, as well as the need to produce an actionable result.
