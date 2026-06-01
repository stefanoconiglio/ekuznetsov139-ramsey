# Understanding Kuznetsov's Ramsey Search Code

This repository accompanies the paper https://arxiv.org/pdf/1505.07186.

The single most important thing to understand is that the search is NOT trying to find forbidden cliques. It is trying to find Ramsey counterexamples.

For R(k,j), a counterexample on N vertices is a coloring with:
- no red K_k
- no blue K_j

For example, for R(4,7), a successful coloring has red clique number at most 3 and blue clique number at most 6.

A red K4 or blue K7 is therefore not a solution. It is evidence that the current branch can never become a solution.

## Big picture

The search space is not the set of all graph colorings.

The paper works with distance colorings. Vertices are arranged in order and all edges having the same distance receive the same color.

Therefore a coloring is represented by a bit string of distances rather than by all edges individually.

The enumeration tree assigns one distance class at a time.

At each node:
- some distances are colored red
- some distances are colored blue
- some distances are still undecided

Undecided edges belong neither to the red graph nor to the blue graph.

Conceptually this is a three-valued state.

## Why pruning is valid

Suppose the current partial assignment already contains a red K4.

No future assignment can remove that K4.

Therefore every descendant of that node also contains a red K4.

Since we are searching for colorings without red K4 and without blue K7, the entire subtree can be discarded.

This is exactly analogous to SAT solving.

## What the code does NOT do

The code does not repeatedly compute:

omega(red)
omega(blue)

Computing maximum clique numbers would be much more expensive.

Instead it asks:

Did the newly assigned distance create a forbidden clique?

This is a threshold question, not a maximum-clique question.

## Incremental clique maintenance

The key optimization is that before a new distance bit is assigned, the current state is assumed valid.

Therefore any newly created forbidden clique must contain at least one edge belonging to the newly assigned distance class.

The code only searches for cliques involving that new distance.

This dramatically reduces work.

## The graph structure

The central data structure is graph.

Important fields:

- mask0 : red distance classes
- mask : blue distance classes
- cliques[] : stored clique lists
- clique_counts[][] : counts by color and size
- parent_cliques[] : references to ancestor clique lists

The surprising feature is that DFS nodes do not copy all clique data.

Instead each node stores newly created cliques and references ancestor clique lists through parent_cliques.

This makes the search much cheaper in memory.

## Partial clique storage

One of the main insights of the implementation is that it explicitly stores clique information.

The search is not rebuilding all clique information from scratch after every move.

When a new distance class is added:

1. Existing clique information is inherited.
2. New cliques containing the newly added distance are generated.
3. Clique counts are updated.
4. Forbidden cliques immediately terminate the branch.

Thus the search carries a growing database of relevant cliques.

## Near-cliques and forced assignments

The paper also uses almost-complete forbidden cliques.

Example:

If a red K4 would be completed by coloring one remaining edge red, that edge is forced blue.

This is the origin of the forced-link propagation discussed in the paper.

The search therefore behaves much more like a CSP solver than a naive graph enumerator.

## set_bit()

set_bit() is the core update routine.

It:

1. Assigns a distance class.
2. Calls graph_set_bit_internal().
3. Searches for new cliques involving the newly assigned distance.
4. Updates clique counters.
5. Rejects immediately if a forbidden clique is found.

## unrolled_flat_search()

This is the main clique-generation engine.

It performs recursive intersection-style clique construction.

Conceptually it:

- starts from the newly added distance
- builds larger monochromatic cliques containing it
- records cliques of useful sizes
- aborts immediately when size k or j is reached

This is why the code never needs a full maximum-clique computation.

## Why clique lists are stored

A natural question is:

How can the program know that it already has five edges of a future forbidden clique?

Answer:

It explicitly stores clique information.

The graph object maintains collections of cliques and clique counts, and descendants inherit these collections through parent references.

The search therefore has direct access to many partial and near-threshold clique structures.

## Mental model

The best mental model is not:

'Enumerate graphs and compute clique numbers.'

Instead think:

'Maintain a partial coloring plus a database of monochromatic cliques and near-cliques. Every new distance assignment updates this database. Forbidden cliques kill branches. Near-cliques create forced assignments. The search continues until either a complete counterexample is found or the tree is exhausted.'

Viewed this way, the paper and the implementation align closely: both are incremental constraint-propagation systems specialized to distance Ramsey colorings.