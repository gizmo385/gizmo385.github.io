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

> Start at the small white node at the bottom of the graph. For each digit \\(d\\) in \\(n\\), follow d black arrows in a succession, and as you move from one digit to the next, follow 1 white arrow.

![The divisibility graph for 7](http://www.tanyakhovanova.com/BlogStuff/Divisibility7.jpg "Wilson's divisibility graph")

Let's explore these graphs by figuring out how to create them ourselves.

## Basic Assumptions

To simply things a little bit, the divisibility graphs that we're going to create are going to
assume that you're determing whether some base-2 number \\(b\\) is divisible by some number \\(n\\).
The first observation that we should make is that we will need at most \\(n - 1\\) nodes in this
graph because each node will be in \\(\mathbb{Z}/\mathbb{Z}\_{n}\\), that is the set of integers
modulus \\(n\\). We will label these nodes \\(q\_{0} \ldots q\_{n-1}\\). Since \\(b\\) is in
base-2, then we know that each node will have 2 edges, each describing how to procede based on a
single binary digit. Since these edges are inherently directional in this problem space, our graph
will be represented as a digraph.

Another key observation that we can make at this point is that the divisibility graph that we're
describing is fundamentally a [finite state
machine](http://www.tanyakhovanova.com/BlogStuff/Divisibility7.jpg). A finite state machine has
a set of states (nodes) and a set of transitions (edges) which tell you which state to move to
based on an input.

Now that we've narrowed down our problem, we can more accurately state it:

> Given a number \\(n\\), construct a finite state machine which accepts the language of bit strings
> which are divisible by \\(n\\).
