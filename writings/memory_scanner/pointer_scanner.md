---
layout: default
title: Creating an Infinite(ish) Depth Pointer Scanning Algorithm
---

---
layout: default
title: Creating an Infinite(ish) Depth Pointer Scanning Algorithm
---

{% include mathjax.html %}

# Creating an Infinite(ish) Depth Pointer Scanning Algorithm
An important problem that has, until now, remained unsolved by memory scanner developers is implementing an efficient pointer scanning algorithm. The premise is simple: we want to find pointer chains from **static memory** to a piece of useful information in **heap** memory.

Existing solutions typically frame this as a graph problem and run into exponential blowup when applying either BFS or DFS. They rely heavily on heuristics to prune the search space rather than changing the problem formulation.

**This paper proposes a range-based reachability method** that avoids the blowup by working in **hop-space**. The algorithm runs in linear time as a function of depth, is memory-efficient, trivially parallelizable, SIMD-friendly, and simple to implement. The key insight: although exponential expansion is inherent to enumerating paths, we can **factor it out** and **defer** it to user time (on-demand expansion) instead of algorithm time.

---

## Problem Definition
Given a target address $T$, find all pointer paths from static memory $S$ (clearly defined below), optionally traversing heap regions $H$, ultimately pointing to $T$. Each hop is constrained by a maximum offset $O$.

For example, $T$ may be player health located at: `[[[game.exe+0x201c]+0x18]+0x24]`
Semantically, this may mean: `World (static +0x201c) -> Player (+0x18) -> Health (+0x24)`

In a high level language:
```
// Static global world instance.
static World;

struct World {
    ...
    Player* player;
}

struct Player {
    ...
    int health;
}
```

**Technical Requirements & Constraints**
- Support large processes (e.g., 16 GB+ RAM).
- Support large depths (e.g., 15+ pointer hops).
- When validating results, missing results are unacceptable. Note that validation may use **admissible heuristics**; it is acceptable to retain an invalid result, but unacceptable to discard a valid result.

**Helpful Observations**
- Scoring and ranking are unimportant for our use cases. Displaying results sorted by static module origin (e.g., `game.exe` vs `phys_x.dll`) and hop distance are sufficient.
- Most process memory is zero. Anecdotally, there are many games where 75% of the process memory bytes are `00`.
- Many pointers are illusory (floats/garbage). Massive path collection up front is futile since validation prunes heavily.
- Pointers are often sparse but aggregated in structures.
- Pointers refer to addresses *near* a thing (e.g., $\(\text{player} + 0x28\)$), which is atypical for classical graph problems, and contributes to the intractability of naive BFS and DFS.
- Displaying *unvalidated* results is rarely helpful; initial sets are huge and noisy.

**Validation workflow (common practice):** relaunch the program, rediscover $T$, and check which collected structures still reach $T$; prune the rest.

**Resiliency goal:** support paths like  
`World* -> Map<ENTITY_ID, Entity*> -> Player* -> Inventory* -> Map<ITEMID, ItemStack*> -> ItemStack* -> ItemCount`,  
despite key swizzling and intrinsic instability. These paths are technically valid when the entity is generally present.

---

## The Algorithm

### Pointer Collection (Range-Based Reachability)
For intuition, we will first ignore heap hops. We want all static pointers from $S$ that land within offset $O$ of $T$. This simplification turns the problem into a very basic binary range search problem.

1. **Snapshot memory.** Take a snapshot of $S$ (static) and $H$ (heap).

2. **Level 0 range:** expand $T$ by $O$ to form  
   $$
   H_0 \equiv (T - O,\; T + O).
   $$
   The key observation is that $T$ exists within $H$, hence naming this $H_0$. Note that negative offsets for pointers are not typical (positive expansion), but supporting this makes us robust to swizzled data structures, which we will cover momentarily.

3. **Static hits at level 0.**  We iterate all static addresses, and collect those that contain a pointer falling within $\(H_0\)$.
   $$
   S_0 = \{ s \in S \mid *s \in H_0 \}.
   $$

At this point, we have all static pointers directly to $T$, but it turns out that with a few additions, we can generalize this algorithm.

4. **Heap hits at level 1.** We perform the same exact operation that we just performed on $S$.
   $$
   H_1 = \{ h \in H \mid *h \in H_0 \}.
   $$

5. **Generalize to deeper levels:**  
   $$
   S_1 = \{ s \in S \mid *s \in \bigcup_{h \in H_1} (h - O,\; h + O) \}.
   $$

6. **Iterate:** for level $\ell \ge 1$, construct heap reachability
   $$
   H_{\ell} = \{ h \in H \mid *h \in \bigcup_{x \in H_{\ell-1}} (x - O,\; x + O) \},
   $$
   And construct static reachability
   $$
   S_{\ell} = \{ s \in S \mid *s \in \bigcup_{x \in H_{\ell}} (x - O,\; x + O) \}.
   $$

7. **Symmetric expansion:** use  
   $$
   (T - O,\; T + O)
   $$
   and similarly for heap expansions.

At completion:
$$
\{S_0,\ldots,S_n\},\quad \{H_0,\ldots,H_n\}
$$
as merged ranges or address lists.

This preserves the **no-exponential** property: we still reason in hop-space. If $\(S_\ell\)$ reaches $\(H_\ell'\)$, it can eventually reach $\(T'\)$.

---

### Deferred Exponentiation (On-Demand Path Materialization)
Enumerating **exact** paths is expensive (and again exponential in the worst case). We **defer** it until the user asks:

- Present a GUI with a tree view:
  - Roots = static modules (e.g., `game.exe`, `phys_x.dll`).
  - Children = top-level heap hops, and so on.
  - Users expand along plausible short-hop branches and stop when they hit $T$.

This is efficient because users rarely need the entire tree; they follow a handful of promising branches. If you *do* need exhaustive automation, the exponent shows up—but **after** validation has already shrunk the search space, which is still better than traditional approaches that pay this cost *before* validation.

---

## Notes on Implementation
- **Data structures:** store merged ranges per level for $\(H_\ell\)$ and optional compact lists for $\(S_\ell\)$. Keep them sorted for binary search.
- **Parallelism:** scanning memory for candidate pointers is embarrassingly parallel; split by region and merge ranges at the end.
- **SIMD:** bulk compare words against range bounds; bitmask results feed into the binary search stage to minimize branches.
- **Heaps vs statics:** collect module boundaries for $S$ (PE/ELF segments). For $H$, use the OS’s VMA enumeration, excluding guard/PROT_NONE.
- **Offsets:** the hop bound $\(O\)$ is a knob; larger $\(O\)$ increases recall (and noise). Symmetric $\((\pm O)\)$ improves robustness to container layouts.
- **Admissible validation heuristics:** only prune when certain an entry is invalid; never prune a potentially valid one.

---

## Summary
By reframing pointer scanning as **level-wise range reachability**, we:
- Avoid exponential path enumeration during collection and validation.
- Leverage cheap binary searches over merged ranges.
- Gain robustness to real-world layouts (maps, swizzled offsets).
- Keep exponentiation **deferred** and **user-driven**.

In short: think in **hops and ranges**, not explicit pointer paths, until you truly need them. And when they are needed, only display the bare minimum to avoid paying the full exponentiation cost.
