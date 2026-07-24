+++
title = "When a Matrix Is the Wrong Data Structure"
date = "2026-07-24"
description = "How adjacency-list thinking helped me debug a sparse binary-matrix calculation"
weight = 1
tags = ["scientific software", "sparse matrices", "data structures", "quantum error correction"]
math = true
+++

I did not come to programming through a computer science background, so I self-studied data structures and algorithms largely because everyone insisted the fundamentals would eventually pay off. This post is about a recent time they did (*not* the *only* time).

I was making a [contribution](https://github.com/Deltakit/deltakit/pull/309) to an open-source quantum error-correction library. The code involved binary matrices over $\mathbb F_2$, where arithmetic is modulo two, so $1 + 1 = 0$. The work was to replace one library with another for a subspace calculation. That subspace represented a logical-operator space. As far as the algorithm I changed in isolation was concerned, the necessary invariants appeared to be preserved.

However, the downstream code expected a particular block-supported basis representation. The new library appeared used a different basis convention, producing an apparently equivalent but unusable basis for the consuming code. That exposed a separate question about hidden conventions in numerical software, which deserves its own post.

## The problem hiding in a basis

A binary matrix is a matrix $B \in \{0, 1\}^{m \times n}$. For example,

$$
\begin{bmatrix}
1 & 0 & 1 & 0 & 0 & 0 \\
0 & 1 & 1 & 0 & 0 & 0 \\
0 & 0 & 0 & 1 & 1 & 0
\end{bmatrix}
$$

Each row is a binary vector. The support of a row is the set of column indices where the entry is `1`. For the rows above, the supports are $\{0, 2\}$, $\{1, 2\}$, and $\{3, 4\}$.


Operationally, the matrix is partitioned at a known column boundary. Every nonzero entry in the relevant basis should lie in the left block. Any support in the right block is a structural violation for the representation the downstream code expects.

$$
A = [f \mid 0].
$$

An arbitrary valid basis and a conventionally structured basis are not necessarily interchangeable in code.

## When printing the matrix stops helping

In tests, I had small fixtures that were easy to inspect in the Python REPL, or in any normal terminal environment. For those, printing the matrix was fine. You can stare at a tiny rectangle of zeros and ones and notice the pattern.

To reproduce the real failure, I had to work with matrices the application was actually choking on. The smallest realistic one I could generate was around `45 × 90`. Which is fine for computers, it's just too large to reason about comfortably by eye, unless you're into that kind of thing.

There is a hidden assumption in "just print the matrix". A dense rectangular printout is itself a choice of representation. It shows all possible row-column relationships, including the ones that do not exist. That is useful when the matrix is tiny. It becomes a poor debugging interface when most entries are zero and the relevant information is the position of the nonzero entries. Two bases may differ in only a handful of rows, while the structural boundary is positional rather than visually obvious.

## Sparse storage already contains the clue

Sparse matrix storage is built around the same observation: do not store every zero if the zeros are mostly uninteresting.

In CSR, or Compressed Sparse Row, entries are grouped by row. An `indptr` array marks where each row begins and ends inside the compressed storage. An `indices` array stores the column index of each nonzero entry. A `data` array stores the corresponding values.

For a binary matrix used only for structural inspection, the `data` array is often the boring part, because every stored value is `1`. The useful information is mostly `indptr` and `indices`: which columns does each row touch?

CSC, or Compressed Sparse Column, applies the same idea by column. It is useful when the natural question is column-oriented: which rows touch this column, which columns are empty, or what is happening in this suspicious right-hand block column by column?

For my investigation, CSR was the more natural conceptual fit. I wanted to treat each basis vector as one object. The basis vectors were rows. I repeatedly needed the support of an individual row. That does not mean CSR is better than CSC. It means the access pattern made CSR feel like the shape of the question.

<details>
<summary>What exactly do CSR and CSC store?</summary>

For a sparse matrix, the usual trick is to store the nonzero values, their indices, and offsets describing where each row or column begins.

For

$$
B =
\begin{bmatrix}
1 & 0 & 1 & 0 \\
0 & 1 & 0 & 1 \\
0 & 0 & 1 & 0
\end{bmatrix},
$$

a CSR-like representation is:

```text
indptr = [0, 2, 4, 5]
indices = [0, 2, 1, 3, 2]
data    = [1, 1, 1, 1, 1]
```

Row `0` uses `indices[0:2]`, giving `{0, 2}`. Row `1` uses `indices[2:4]`, giving `{1, 3}`. Row `2` uses `indices[4:5]`, giving `{2}`.

The `data` array stores the values at those positions. In a general sparse matrix, those values matter. In a binary structural check like this one, the fact that an entry exists is often the main point, so the indices carry most of the debugging information.

CSC stores the same kind of information, but grouped by column rather than row. That makes column traversal cheap in the same way CSR makes row traversal cheap.

</details>

## From a matrix to a bipartite relation

There is also a graph hiding in the matrix, but it is worth saying that carefully.

A binary matrix can be viewed as a bipartite incidence relation. One set of vertices represents basis rows. The other set represents matrix columns. A `1` at $B_{ij}$ represents an edge between row vertex $i$ and column vertex $j$.

For the small example in the collapsible section, this gives:

```text
row 0 -> columns 0, 2
row 1 -> columns 1, 3
row 2 -> column 2
```

That is conceptually close to an adjacency-list representation. Each row stores its neighbouring columns. The dense matrix stores all possible relationships, including absent ones. The sparse or adjacency-style view stores only the relationships that exist.

CSR is not identical to an adjacency list. It is an array-based compressed representation of essentially the same neighbourhood information. But once I thought of the matrix this way, the debugging problem stopped looking like "compare two rectangles" and started looking like "compare two sets of neighbourhoods".

## The algorithms were ordinary

The actual algorithms I used were not exotic. That was the nice part.

First, extract the support of every row:

$$
S_i = \{j : B_{ij} = 1\}.
$$

For a dense NumPy array, that can be as simple as:

```python
supports = [
    tuple(numpy.flatnonzero(row))
    for row in matrix
]
```

For CSR, it is the same idea without scanning the zeros:

```python
supports = [
    tuple(matrix.indices[matrix.indptr[i]:matrix.indptr[i + 1]])
    for i in range(matrix.shape[0])
]
```

Then canonicalise. Each row support is sorted and converted to an immutable tuple. That gives a stable exact representation for equality comparisons, dictionary keys, set membership, and grouping. The tuple is the canonical key; Python may hash it internally, but the hash is not the mathematical identity. The exact support tuple is.

With two candidate bases, the comparison becomes standard set intersection and set difference:

```python
common = expected_rows & actual_rows
missing = expected_rows - actual_rows
unexpected = actual_rows - expected_rows
```

There is an important caveat here. This compares exact basis vectors as represented by rows. Two different row sets may span the same subspace. So this is a debugging comparison of representations, not a general test of subspace equality.

The block-boundary validation was similarly ordinary. Given a block boundary $k$, detect whether any row support crosses it:

$$
\exists j \in S_i \text{ such that } j \geq k.
$$

Written directly:

```python
violating_rows = {
    row_id: support
    for row_id, support in enumerate(supports)
    if any(column >= block_boundary for column in support)
}
```

And because my supports were sorted, the check could be reduced to the last element:

```python
violating_rows = {
    row_id: support
    for row_id, support in enumerate(supports)
    if support and support[-1] >= block_boundary
}
```

That second version only works because the support is sorted. Without that invariant, the final column in the tuple would not tell you anything special.

## Turning support into a table

Another useful view was to flatten the matrix support into records:

```text
basis, row, column
```

That is just an edge list, or a relation, depending on which language you prefer. Once the data looks like that, the question becomes a grouping and filtering problem.

For example:

```python
rows, columns = numpy.nonzero(matrix)

support = pandas.DataFrame({
    "row": rows,
    "column": columns,
})

violations = support.loc[
    support["column"] >= block_boundary
]
```

Pandas was not required. SQL would have been fine. Polars or DuckDB would also have been fine. The important step was not the tool. The important step was turning the sparse matrix into the relation I actually wanted to query.

## Tools I could have used

I rolled a small version of this tooling myself because the immediate problem was narrow and I wanted direct control over the output. The data was modest. The representation needed to match the debugging question. I wanted a transparent mapping between rows, columns, and violations. Introducing a larger graph abstraction would have been unnecessary for the first diagnosis.

Still, recognising the standard representations matters because it tells you when to stop rolling your own as the analysis grows.

For sparse extraction and storage, NumPy gives you `numpy.nonzero`, `numpy.argwhere`, and `numpy.flatnonzero`. SciPy gives you `scipy.sparse.csr_array`, `csr_matrix`, `csc_array`, `csc_matrix`, and `scipy.sparse.find`.

For graph-oriented analysis, SciPy's `scipy.sparse.csgraph` can operate directly on sparse matrix representations, and NetworkX gives a more explicit graph object if that is the level of abstraction you need.

For relational comparison, pandas is the obvious Python hammer, but Polars, DuckDB, and SQL all fit the same pattern: represent the support as rows in a table, then group, join, filter, and compare.

## What the new representation exposed

The new representation exposed the mismatch directly. I could see which row supports were common to the expected and actual outputs, which rows were missing, which rows were unexpected, and which supports crossed the block boundary.

That gave me something concrete to take back to the review thread: the candidate implementation appeared to produce a plausible space, but not the block-supported representation the downstream code had learned to expect. I posted the hypothesis and the design question there. The final basis calculation remains under discussion with the maintainers.

It also suggested a deeper issue, which I am not going to fully unpack here. The previous dependency may have been returning a basis with an undocumented convention that downstream code had learned to rely upon. Reproducing that convention and removing the downstream dependency are separate design questions. That deserves another post.

## The broader lesson

The new representation did not solve the wider basis-construction problem by itself. It made the failure precise.

I could not reason effectively about a realistic `45 × 90` binary matrix by inspecting it, so I changed its representation. The mathematical object stayed the same, but the debugging interface changed completely. Dense rows became supports. Supports became canonical tuples. Tuples became set operations. Nonzero entries became table records. The block condition became a filter.

I had started by staring at a matrix. I ended with relational queries.

You can also read [about me](../../about/) or reach me through the [contact page](../../contact/).
