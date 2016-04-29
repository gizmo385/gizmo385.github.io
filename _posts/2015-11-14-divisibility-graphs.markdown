---
layout: post
title: Divisibility Graphs with Loom and Clojure
categories: [clojure, programming]
tags: [functional, lisp, clojure, programming, graphs]
description: A short study of divisibility graphs with Clojure and Loom
comments: true
fullview: true
---

## Introduction

Recently, I stumbled on a post which is several years old, but caught my attention. In
[the post](http://blog.tanyakhovanova.com/2009/08/divisibility-by-7-is-a-walk-on-a-graph-by-david-wilson/),
David Wilson discusses how you can express the process of determing if a base-10 number is
divisible by 7. The most important element of the post is this:

> Start at the small white node at the bottom of the graph. For each digit \\(d\\) in \\(n\\),
  follow \\(d\\) black arrows in a succession, and as you move from one digit to the next, follow
  1 white arrow.

![The divisibility graph for 7](http://www.tanyakhovanova.com/BlogStuff/Divisibility7.jpg "Wilson's divisibility graph"){:width="300px"}

The method described by Wilson uses a graph where you follow a particular sequence of edges
based upon the current digit you're looking at. If you're number eventually results in you
returning to the start state, then the number is divisible by 7. We will consider a simplification
of these graphs and create a representation in Clojure using the graph library
[Loom](https://github.com/aysylu/loom).

## Breaking down the problem

First of all, let's simplify this a little bit. For the sake of conciseness, we will be restricting
our divisibility graphs to base-2 integer. Realistically, this is not much of a restriction, since
we easily translate a base-10 integer into its corresponding base-2 representation. With this
simplification, let's make some quick observations about the problem:

1. The divisibility graph of some number \\(n\\) will contain exactly \\(n\\) nodes. These nodes
   represent the current remainder when computing divisibility.

2. Since we're dealing with bit strings, each node will have exactly 2 edges, namely 1 or 0. We can
   leverage the properties of how binary numbers are constructed to make these edges easily to
   generate.

3. Our definition of divisibility under this scheme will be that if it possible to traverse the
   graph of \\(n\\) with some number \\(k\\) and return to the node labeled 0, then \\(k\\) is
   divisible by \\(n\\).

Taken all together, these observations lead us to discover that the divisibility graph of a number
\\(n\\) is simply an \\(n\\) state finite state machine whose start state and final state are the
same with state transitions on a 0 or a 1.

## Creating our "Graph"

Thus far we've described these solely as graphs, but as a first step maps should do a pretty good
job of modeling this system. We know that we need \\(n\\) "nodes", numbered \\(0 \cdots (n - 1)\\),
but we need to figure out how we're going to be labeling our "transitions". To do so, we will use
this piecewise function to describe how each transition will get labeled:

$$
T(n, b) =
\Bigg\{
\begin{array}{cc}
    \begin{array}{cc}
      2n \bmod n        & b = 0 \\
      (2n + 1) \bmod n  & b = 1
    \end{array}
\end{array}
$$

The function \\(T(n, b)\\) represents the transition out of the state labeled \\(n\\) on some bit
\\(b\\) where \\(b \in \\{0, 1\\}\\). The justification relies on how a number is encoded as a bit
string. When an additional zero is added to the end of a bit string, it has the effect of doubling
the value (consider 01 vs 010). Conversely, when a 1 is added to the end of a bit string, it has
the effect of doubling the value and adding 1 to it (consider 01 vs 011).

This function allows us to define our graph as a map where the keys are the node labels, and the
values are 2 entry maps which define the transitions based on the results of \\(T(n, b)\\). So how
might that look in Clojure?

```clojure
(defn divisibility-map
  "Creates the divisibility map for some number n"
  [n]
  (zipmap
    (range n)
    (for [i (range n)]
      {0 (mod (* 2 i) n)
       1 (mod (inc (* 2 i)) n)})))
```

Wait... that's it? That encapsulates our graph? Yep! Let's look at the result of creating
a divisibility map for the number 5:

```clojure
user=> (divisibility-map 5)
{0 {0 0, 1 1},
 1 {0 2, 1 3},
 2 {0 4, 1 0},
 3 {0 1, 1 2},
 4 {0 3, 1 4}}
```

It should be noted that the modulus operators both in the original function \\(T\\) and in the
Clojure code are to protect against wrap around.
