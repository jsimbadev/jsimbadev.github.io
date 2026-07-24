+++
title = "Datastructures and Algorithms for Scientific Programming"
date = "2026-07-24"
description = "An account where adjacency lists actually helped debugging numerical software"
weight = 1
tags = ["research", "linear algebra", "datastructures", "algorithms"]
+++

Not coming from a computer science background, I've had to pick up data structures and algorithms on the job, and not because I needed them directly, but because I took the advice from the consensus and just did (not that I'm _finished_) it.

This post is just an anecdote where the fundamentals, as boring and far removed as they seem, came in handy. 

This is *NOT* the only time those same fundamentals have helped me directly, or guided decisions spiritually, but I thought this particular instance was interesting because of the context.

I was working on a piece of computational linear algebra code calculating a pariticluar span of some operator space. This involved working with binary matrices (TODO show mathematical definition of binary matrix, with an example). My implementation returned a valid subspace, by the mathematical constraints, however, the client code of the subspace required a specific block structure. 

In tests I used small matrix fixtures that were easy to inspect in the python REPL, or any typical terminal environment. However, the client code was still throwing errors due to the basis representation not having the necessary block structure.

In order to reproduce the error I had to work with real matrices that the application was choking on, which the smallest I could generate was around `45x90`... and comparing two of these by eye is not fun, unless your into that kind of thing.

But naturally matrices are tabular, and even better sparse matrices are routinely compressed with CSR or CSC data structures, which themselves are matrix -> graph transformations. 

The structure I was looking for was 

$$
A = [f | 0]
$$

but my implementation was not adhering to this.

Using invariants like basis being linearly independant meaning each row was unique meant I could take hashes of the entire row and this yielded a unique basis identifier, so I could do the same transformation over different binary matrices to find idendentical basis vectors, and use standard set logic to find overlap, and unique basis vectors.

More importantly, due to the spare nature of the matrices, I could drastically reduce the number of values required to represent a row, (row,  position_of_number 1) is sufficient to describe each row, stack these and one gets a view of the original matrix.

The question of whether the generated basis satisfy the block structure is a standard relational. Do it by pandas, sql or any other means.

It amounted to checking the value of the largest position of 1, and if each row shared this bondard 1 value.  

You can also read [about me](../../about/) or reach me through the [contact page](../../contact/).
