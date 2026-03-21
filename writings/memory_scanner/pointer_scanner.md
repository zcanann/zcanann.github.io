---
layout: default
title: Defeating Exponential Blowup - A Novel Approach to Pointer Scanning
---

{% include mathjax.html %}

# Defeating Exponential Blowup - A Novel Approach to Pointer Scanning
Efficient pointer scanning is a longstanding problem in reverse-engineering, where reconstructing pointer paths can be crucial to understanding or exploiting a program.

Until now, every implementation suffers from exponential blowup from applying DFS or BFS to the problem. Generally, some form of backwards search from a target address to static roots is performed.

This paper shows that exponential blowup can be avoided by reframing the problem around the actual user requirements. This is accomplished by effectively factoring out the exponential portion of the problem and deferring it to user-time.

The core idea behind our algorithm is to stop treating "pointer path" as the primitive. The primitive is **reachability by hop**. We collect and validate in **hop-space**, not pointer-space. We only pay the explicit path cost later, on demand, when the user actually expands and explores collected pointers.

This reframing allows pointer collection and validation to happen over a reachability graph, rather than a pointer graph, avoiding exponential materialization costs.

## Problem Statement
We want to find pointer chains from static memory to some target.

Let:
- \(S\) be the set of static pointer locations.
- \(H\) be the set of heap pointer locations.
- \(T\) be the target address set.
- \(O\) be the maximum allowed offset radius.

For a classic address-seeded scan, \(T\) contains one address.

For a value-seeded scan, \(T\) contains **every address currently holding that value under the chosen data type**. That matters because the exact same collection pipeline can then run on either mode. The only difference is how we build the initial target set.

The goal is to find chains like:

```text
[[[game.exe+0x201C]+0x18]+0x24]
```

Semantically:

```text
World -> Player -> Health
```

Or in code:

```cpp
static World* g_world;

struct World {
    Player* player;
};

struct Player {
    int health;
};
```

The important subtlety is this: pointers usually do not point to the exact field we care about. They point near it. So each hop is not checking equality. It is checking whether a pointer value falls within a tolerated range around the next target.

## Why Naive Enumeration Blows Up
Suppose every candidate node fans out to \(b\) more candidates, and we search to depth \(d\).

Naively, the search space behaves like:

\[
1 + b + b^2 + \dots + b^d
\]

Which is:

\[
O(b^d)
\]

That is not a "bad implementation" problem. That is the natural cost of eagerly enumerating paths.

And the insult to injury is that most of those paths are garbage:
- Integer data pretending to be pointers.
- Float bit patterns that happen to look pointer-ish.
- Stale heap structure.
- Completely unstable chains that fail immediately on validation.

So the old way pays exponential cost up front to generate data that you mostly did not want.

## The Key Reframing: Work in Hop-Space
Instead of asking:

> what are all explicit pointer paths to the target?

We ask:

> which pointer locations can reach the current frontier within one hop?

Define the expansion of a target set \(F\) by radius \(O\) as:

\[
R(F) = \bigcup_{x \in F} [x - O,\; x + O]
\]

In practice we do not store this as an absurd list of tiny windows if many overlap. We sort the target addresses, expand them, and merge overlapping ranges into a compact frontier range set.

That gives us a very clean level recurrence.

## Initial Collection
Let \(F_0 = T\).

For each level \(\ell\):

\[
S_\ell = \{ s \in S \mid *s \in R(F_\ell) \}
\]

\[
H_\ell = \{ h \in H \mid *h \in R(F_\ell) \}
\]

Then the next frontier is simply:

\[
F_{\ell+1} = \text{addr}(H_\ell)
\]

Where \(\text{addr}(H_\ell)\) means the addresses of the heap pointers we just found.

That is it. That is the whole shape. We do **not** enumerate paths during collection. We just discover which static and heap nodes are reachable at each hop depth.

### Scalability
At level 1, we find everything that can point near the target.
At level 2, we find everything that can point near level 1 heap nodes.
At level 3, we find everything that can point near level 2 heap nodes.

And so on.

The amount of work scales with:
- The amount of memory scanned.
- The number of levels requested.
- The compactness of the frontier range set.

This is substantially better than scaling with the number of explicit pointer paths hiding inside the graph.

Additionally, because process memory is bounded and has a typical program structure, the algorithm "feels linear". Experimentally, beyond high depths the cost of each level remains constant.

## Collection Algorithm
Our algorithm is intentionally shaped like a simple pipeline:
1. Collect snapshot values.
2. Discover pointer levels.
3. Build a retained session.
4. Lazily materialize explicit display nodes later.

This separation matters. The collection stage is CPU-bound kernel work. The session stage is just retained structure. The tree UI stage is deferred exponentiation.

### Step 1: Collect values
Statics and heaps are stored as snapshots. We refresh their bytes first so the pointer scan kernels are working against current data.

### Step 2: Build a frontier range set
Given target addresses \(T\), we expand each by \(\pm O\), sort them, and merge overlaps.

So if the target set is:

\[
\{0x3000, 0x3080, 0x4000\}
\]

and \(O = 0x100\), then instead of three separate windows we get two merged ranges:

\[
[0x2F00, 0x3180], \quad [0x3F00, 0x4100]
\]

This matters because the kernel is now checking pointer values against a compact ordered range structure, not a noisy list of redundant intervals.

### Step 3: Split memory into typed scan tasks
The pointer scan does not classify static-vs-heap after a match. That would be wasted hot-path work.

Instead, the snapshots are cut into **scan tasks** up front:

- static tasks already know their `module_index` and `module_base`,
- heap tasks already know they are heap,
- task boundaries keep a small pointer-width overlap so we do not miss matches near chunk edges.

This is important because it keeps the inner loop honest. When a kernel reports a pointer match, the code already knows whether it belongs to a static or heap candidate. No per-hit module walk. No ad hoc post-classification. Just append the candidate.

### Step 4: Run the search kernel
The range search kernel picks a search mode based on frontier shape.

Very small frontiers can just use scalar linear checks.
Larger frontiers switch to binary checks over merged ranges.
There is also a SIMD-friendly linear kernel for cases where chewing through aligned pointer lanes wins.

Conceptually, the hot loop is just:

\[
\text{keep pointer at address } a \iff *a \in R(F_\ell)
\]

And because the region tasks are embarrassingly parallel, this scales very naturally across cores.

### Step 5: Retain levels, not paths
For each depth we retain two sorted candidate lists:

- static candidates for that level,
- heap candidates for that level.

Static candidates across all discovered depths become potential roots.
Heap candidates become the next frontier source.

This is the point where a lot of pointer scanners go wrong. They start eagerly rebuilding trees here.

That is unnecessary.

We only need levels.

## Deferred Exponentiation
Eventually the user wants to browse actual chains. Fine. That is where we pay the path cost. But we pay it **late** and when it does not matter nearly as much.

The session keeps per-level candidates and lazily materializes display nodes when the user expands part of the tree. A root page is basically all retained static candidates, ordered by shorter chains first. Expanding a node does a bounded lookup into the next relevant level and only materializes the requested page.

This means the algorithm does **not** build the full tree up front. That is deliberate.

The graph may contain a huge number of possible paths. Most users are going to expand a tiny fraction of them:
- Shortest paths first.
- The most plausible module first.
- Then a few children.
- Then they stop when they find what they want.

So the system is built around that workflow. The exponential part of the problem is still real, but most users simply do not experience it with this algorithm and workflow.

## Validation Algorithm
Validation is where most pointer scanners become either painfully slow or conceptually incoherent.

The design goal of our algorithm is simple:

**validation should look like collection as much as possible.**

That is exactly what the current validator does.

Suppose we restart the target process and rediscover a new target set \(T'\). Build:

\[
F'_0 = T'
\]

Then for each level:

1. rebuild the live heap frontier from the validation snapshot,
2. reread the stored static roots for that level,
3. keep statics whose live pointer value still reaches the current frontier,
4. use rebuilt heap addresses as the next frontier.

More concretely:

\[
H'_\ell = \{ h \in H' \mid *h \in R(F'_\ell) \}
\]

\[
S'_\ell = \{ s \in S_{\ell,\text{stored}} \mid *s_{\text{live}} \in R(F'_\ell) \}
\]

\[
F'_{\ell+1} = \text{addr}(H'_\ell)
\]

### Static Rebasing
Stored static candidates are not kept as raw absolute addresses alone. They also retain module identity plus module offset.

So during validation, a static candidate from:

```text
game.exe + 0x10
```

can be rebased to the module's **current** load address before rereading memory.

This is crucial because ASLR is not a special case here. It is the expected case.

### Conservative Validation
This validator is intentionally a **hop-space** validator, not an exhaustive pointer-space proof.

That means it is designed to avoid false negatives. If a retained static still reaches the rebuilt frontier at its level, we keep it.

That is a stronger guarantee in the direction we actually care about:

- accidentally dropping a valid chain is bad,
- retaining an extra chain is tolerable.

This is especially relevant for value-seeded scans. If you scan for one value, then validate against another, the validation pass can still conservatively retain chains that an exact fresh pointer-path enumeration might reject.

That is not a bug in the mathematical framing. That is the expected tradeoff of conservative hop-space validation.

If you want the shortest description, it is this:

**validation is a conservative pruning pass over the retained reachability structure, not a full fresh path proof.**

## How This Avoids Exponential Blowup During Collection
Collection and validation both scale level-by-level.

Ignoring the eventual user-driven path materialization cost, the core reachability pass behaves like:

\[
O(D \cdot W)
\]

Where:

- \(D\) is the requested depth,
- \(W\) is the work to scan the relevant memory through the chosen kernels.

The important thing is what is **missing** from that expression: there is no branching factor term from explicit path expansion.

That does not mean exponentiation vanished from the universe. It means we stopped paying it during the wrong phase.

## Value-Seeded Pointer Scans
Value pointer scans slot naturally into this formulation.

Instead of starting from one target address, we first resolve the target value under the chosen data type and collect every matching address:

\[
T = \{ a \mid \text{value}(a) = v \}
\]

After that, the pipeline is exactly the same.

That is a nice property of the formulation. The collection engine does not care whether the frontier came from:

- one address,
- many addresses,
- or a value scan that exploded into many live targets.

A frontier is a frontier.

## Practical Implementation Notes
A few implementation details matter more than they might seem.

### 1. The inner loop is typed before it starts
Static and heap tasks are split before scanning, so the hot path is not doing classification work after every hit.

### 2. Task boundaries overlap by pointer width
This prevents chunking from dropping valid pointers that begin near the end of one task and extend into the next.

### 3. Frontier ranges are merged aggressively
If many target expansions overlap, we collapse them before scanning. This shrinks the effective search structure and makes the kernel's life much easier.

### 4. Session data stays level-based
The retained structure is levels plus candidates. Not rebuilt trees. Not memo junk. Just the data actually needed for lazy expansion.

### 5. Tree nodes are materialized late
Display nodes carry things like:

- resolved target address,
- current display depth,
- total branch depth,
- pointer offset for that specific hop.

That is UI-facing structure, not scan-facing structure.

Keeping those concerns apart is what keeps the collector fast and the browser understandable.

## The Mental Model
If I had to compress the whole design into one sentence, it would be this:

**Pointer scanning is not about discovering paths first. It is about discovering reachable hop frontiers first, then paying for paths only when someone actually cares.**

Once you internalize that, the system stops feeling magical.

At each level, we are just doing:
- Frontier range construction.
- Pointer-value search over memory.
- Static/heap retention.
- Next frontier generation.

That is a very sane systems problem. And it turns a historically ugly exponential algorithm into something that is actually pleasant to run, validate, and browse.

## Summary
The current Squalr pointer scanner works because it takes the expensive part of the problem and moves it to the only place where paying it makes sense.
- Collection works in hop-space.
- Validation works in hop-space.
- Static roots are retained by level.
- Heap frontiers are rebuilt level-by-level.
- Exact display trees are materialized lazily.
- Value-seeded scans fit the same model as address-seeded scans.

In short, **collect reachability, not paths.** Then let the user expand the parts of the graph that actually matter.
