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

## The Problem

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

## Divisibility Maps

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
the effect of doubling the value and adding 1 to it (consider 01 vs 011). It should be noted that
the inclusion of the \\(\bmod\\) operator is to protect against wrap around.

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

And that defines our entire map? Yep! Let's look at the result of creating a divisibility map for
the number 5:

```clojure
user=> (divisibility-map 5)
{0 {0 0, 1 1},
 1 {0 2, 1 3},
 2 {0 4, 1 0},
 3 {0 1, 1 2},
 4 {0 3, 1 4}}
```

Now that we have this map, we can investigate traversing it to determine if a number is divisible by
\\(n\\).

## Defining Divisibility

We will be expressing divisibility in terms of modulus, so to do that, we need to figure out how we
can effectively express modulus in this scheme.

1. To determine if some integer \\(k\\) is divisible by \\(n\\), we need to convert \\(k\\) to its
   binary representation. We can do this with `Integer/toBinaryString`, documented
   [here](https://docs.oracle.com/javase/8/docs/api/java/lang/Integer.html#toBinaryString-int-).

2. Once we've converted it to a bit string, we need to retrieve each individual bit as an integer.
   For the sake of simplicity and clarity, we can do through use of `map`, `str`, and
   `Integer/parseInt`.

3. Lastly, once we have a list of the bits in the bit string, we need to figure out some way to
   encapsulate our state transitions and return our result. This can be easily expressed as
   a [reduction](https://clojuredocs.org/clojure.core/reduce).

```clojure
(defn modulus
  "Returns the result of taking k mod n. The argument n should be a divisibility map."
  [k n]
  (let [bits (map (fn [i] (Integer/parseInt (str i))) (Integer/toBinaryString k))]
    (reduce (fn [state bit] (get (get n state) bit)) 0 bits)))
```

The binding in the `let` statement performs the first two steps above. After we have the list of
bits, we can then reduce the list by keying into our divisibility map with our current state and the
current bit. The result of this reduction will be the last returned state in the reduction. For
example:

```clojure
user=> (modulus 25 (divisibility-map 5))
0
user=> (modulus 27 (divisibility-map 5))
2
user=> (modulus 325 (divisibility-map 7)) ; Verify this one on your own
3
```

Now that we have something that can effectively compute \\(k \bmod n\\), our divisibility predicate
should follow naturally:

```clojure
(defn divisible?
  "Returns whether or not the integer k is divisible by the integer n"
  [k n]
  (zero? (modulus k (divisibility-map n))))
```

So now that we've explored this problem and can easily generate these divisibility maps, let's use
these techniques to generate some awesome visualizations of the concept!

## Divisibility Graphs

[Loom](https://github.com/aysylu/loom) is a Clojure framework that allows for the easy creation of
graphs. Using Loom, and the lessons we learned from generating our divisibility maps, we can also
generate Loom graphs:

```clojure
(defn divisibility-graph
  "Creates a directed graph to encapsulates the structure of the divisibility map for an integer n"
  [n]
  (let [zero-transitions (map (fn [i] [i (mod (* 2 i) n)])       (range n))
        one-transitions  (map (fn [i] [i (mod (inc (* 2 i)) n)]) (range n))]
    (as-> (loom.graph/digraph) g
      (apply loom.graph/add-edges g zero-transitions)
      (apply loom.graph/add-edges g one-transitions)
      (loom.attr/add-attr-to-edges g :label 0 zero-transitions)
      (loom.attr/add-attr-to-edges g :label 1 one-transitions))))
```

There's a lot going on in this function, so let's break it down.

* The let bindings create the graph's edge definitions. There are two, one for edges labled 0 and
  another for edges labeled with 1. Each tuple contains 2 node ids, which are the current node and
  node the edge to go to respectively.

* After defining our edges, we can create a directed graph and then add the edges. We also add
  labels to our edges. The reason for this will be made clear in a moment.

Now that we have an actual digraph, what can we do with it? We've already expressed divisibility and
the modulus operation in terms of our divisibility map, so what bonuses does this graph
representation afford us? Well luckily, Loom is able to output the graph in a
[Graphviz](http://www.graphviz.org/) format and generate an image of our graph!

Let's consider the divisibility graph of 5 once again:

```clojure
user=> (divisibility-graph 5) ; A raw call to our divisibility-graph function
{:nodeset #{0 1 4 3 2},
 :adj {0 #{0 1}, 1 #{3 2}, 2 #{0 4}, 3 #{1 2}, 4 #{4 3}},
 :in {0 #{0 2}, 2 #{1 3}, 4 #{4 2}, 1 #{0 3}, 3 #{1 4}},
 :attrs
 {0 {:loom.attr/edge-attrs {0 {:label 0}, 1 {:label 1}}},
  1 {:loom.attr/edge-attrs {2 {:label 0}, 3 {:label 1}}},
  2 {:loom.attr/edge-attrs {4 {:label 0}, 0 {:label 1}}},
  3 {:loom.attr/edge-attrs {1 {:label 0}, 2 {:label 1}}},
  4 {:loom.attr/edge-attrs {3 {:label 0}, 4 {:label 1}}}}}
user=> (view (divisibility-graph 5)) ; This should make the image of the graph appear
```

The second command should generate something similar to the following:

![Our generated divisibility graph of 5](/public/images/DivisibilityGraph5.png "Divisibility graph of 5"){:height="300px"}

We can readily generate these for even large numbers! For example, here is the divisibility graph of
50. If you would like to give it more attention, click on the image for the full resolution.

[![Divisibility graph of
50](/public/images/DivisibilityGraph50.png "It's a bit hard to read!"){:height="500px"}](/public/images/DivisibilityGraph50.png)

## Conclusion

Overall, examining this simple problem allows us to see how expressive Clojure is as a language. We
are able to epxress many complex structures in terms of simple lists and maps, which Clojure handles
easily. It's also a bit of good fun to dig through these and make some interesting visualizations!
