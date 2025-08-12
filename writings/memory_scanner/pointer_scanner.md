# Creating an Infinite(ish) Depth Pointer Scanning Algorithm
An important problem that has, until now, remained unsolved by memory scanner developers is implementing an efficient pointer scanning algorithm. The premise of the problem is simple, we want to find pointer chains from static memory to a piece of useful information in heap memory.

The existing solutions all fall into the same trap of framing this problem as a graph problem, resulting in exponential blowup when attempting to apply BFS or DFS to pointer discovery. As a result, these algorithms rely heavily on heuristics to cut down the search space, without solving the underlying nature of the problem.

In this paper, I propose a novel method of reinterpreting the problem as a range based reachability problem, completely avoiding the exponential blowup by solving the problem in linear time as a function of depth. This solution is extremely memory efficient, multi-thread compatible, SIMD friendly, and simple to implement.

The key insight is that while it is technically impossible to avoid the exponential blowup, it is possible to factor it out of the problem. We can then perform the expoentiation much later in the process when the cost is substantially lower, and defer it to user-time rather than algorithm-time, making the exponential cost invisible to the user.

# Problem Definition
To formally state the problem, given an address {T}, find all pointer paths from {S} (static memory, which will be defined clearly later), that optionally traverse through {H} (heaps, or non-static memory), ultimately pointing to {T}. Each hop is constrained by a maximum offset {O}.

For example, the health of the player in a game may be at [[[game.exe+0x201c]+0x18]+0x24]. This may semantically translate to:
World (Static offset 0x201c) -> Player (offset 0x18) -> Health (offset 0x24).

Missing results is unacceptable outside of pruning O. We want to consider all paths, even ones that are not semantically sound (ie if a pointer goes from UI* -> Player*, this is fine, even if World* -> Player* is more correct).

We should leverage some real world constraints to better understand the problem:
- The algorithm must work for large games (ie 16GB+ RAM).
- We want to be able to support large depths. (ie 15 pointers deep, or perhaps even much larger).
- Most memory in a process is zero (gut estimate for most games is 75%). Pointers are actually somewhat sparse, but can be aggregated in certain data structures. However, with that said, there are still enough pointers that a basic DFS or BFS becomes intractable and exponentially explodes.
- Many pointers are illusory, coming out of float values, garbage data, or happenstance. This could mean that up-front massive collections of paths is a futile effort given that many would be filtered in validation steps.
- Scoring and ranking are simply not important. Sorting by least number of hops, and displaying information about which static base (ie 'game.exe' vs 'phys_x.dll') in a UI is often sufficient for an experienced reverse-engineer.
- Displaying unvalidated results is almost never helpful. Until a validation round has been performed, the initial set of pointers is nearly guaranteed to be too large and filled with too many false positives.
- The nature of this problem is not actually fully compatible with standard graph approaches. It is atypical for graphs to point "near" something, as pointers do. ie [player+0x28]. This warrants out-of-the-box thinking.

Validation is generally performed by relaunching the program, finding T again, and somehow validating whatever collected data structures are still able to reach T, pruning those that cannot. That said, an often overlooked and critical piece of information is that validation can use admissible heuristics (ie never pruning valid results, but its okay to not prune an invalid result).

Additionally, it would be nice to be resilient to structures like World* -> Map<ENTITY_ID, Entity*> -> Player* -> Inventory* -> MAP<ITEMID, ItemStack*> -> ItemStack* -> ItemCount. Resiliency meaning capable of discovering these pointer paths.

This is remarkably challenging because the maps can swizzle offsets, and these paths are intrinsically non-stable. But it would be actually useful if we could find and keep these paths in some way, because they are technically valid, especially if the entity is generally within the map, like a Player*.

# The Algorithm

## Pointer Collection
For a moment, let us forget about the constraint of hopping through H. This means we simply want to find all pointer paths in S -> T. First, we take a snapshot of all static and heap memory, S and H. This is required for all pointer scanning algorithms.

It turns out that this can be expressed as a simple range check problem. We expand T backwards by O, giving us (T-O, T). In fact, T exists within H, so we will call this expanded range { H_0 }. Then we check every single address in S to determine if it falls within H_0. This will give us a list of all static pointers that fall within O of T. We will call this set S_0.

This entirely solves the problem for the case where we skip hopping through H, but it turns out that we can generalize the solution with a few additions. If we do the same operation on the heap, we can collect all addresses in the heap that fall within O of T. In other words, we collect all addresses in H that fall within H_0, yeilding H_1.

This is where the pattern starts to generalize. If we want to find static addresses that hop through the heap 1 time to reach T, we simply iterate over S, and check the value at each address to see if it falls within O of every address in H_1 via a binary search over the sorted heap ranges. This can be done by first expanding all addresses collected in H_1 by O. Care must be taken in the implementation to merge any regions that overlap from this operation. Everything that matches can be inserted into S_1.

At this point, the entire algorithm is completely generalized to any depth. At each level, we produce S_l and H_l by finding all pointers that fall within the previous H_l-1. The first case of course is the case where H_0 = {(T-O, T)}.

With this new binary search part, we now have developed a level based range reachability algorithm. In fact, we can augment this further to expand by O in both directions, creating (T-O, T+O), and doing similar for heap expansions. This gives us some resiliancy to map-like structures, which can shuffle offsets.

It is important to note that we are not concerned with collecting exact offsets -- the key insight of this algorithm is operating in hops as the fundamental unit. This is how we avoid exponential blowup. At no point do we evaluate any individual path, instead we only check for reachability. We are answering the question "can we hop to T through S and H at each level by a maximum hop size of O".

Once our algorithm is done, we have a set of {S_0, ... S_n}, {H_0, ... H_n}. These are very lightweight data structures that only contain ranges.

## Pointer Validation
Most of our results will be spurious, due to false pointers from garbage data or floats that were wrongly interpreted as pointers. To validate this, the recommended method is to relaunch the target program, rediscover T manually through memory scanning or other means. At this point, we can consider every previously collected H_l to be useless.

So, we discard these entirely, and rebuild them using the new (T-O, T+O) as our H_0. We do not rebuild S_l levels, as these are considered to still be valid. Once we rebuild each H_l using the exact method as before, we can validate each level of S_l by checking each static address at each level to see if it still contains a pointer that falls within H_l. Note that we only check if it falls within the exact level, ie S_0 checks to see if the pointer falls within H_0, and S_1 only checks if the pointers fall within H_1. Again, this is validated with a binary search, and if the pointer does not fall within range, it is discarded.

The key efficiency here is that we are still avoiding exponential blowup. We are making weaker guarantees in hop-space rather than pointer-space. If S_l can still reach H_l, then it is guaranteed to be able to eventually reach T.

## Deferred Exponentiation
Finding these exact paths is expensive, and requires walking backwards form T, but the nice part is that we can defer this expensive operation. We actually never need to construct the exact paths until the user requests them. One possible UX for this flow would be to have a GUI with a tree based expansion. The roots would be each static module, ie 'game.exe' or 'phys_x.dll'. Then these can be expanded further to see all top level heap hops. These can be expanded recursively until they hit T. Generally, the user will greedily expand the shortest hops in the most relevant looking static module.

This would be prohibitively expensive if we wanted to walk the entire tree, but the key insight is that for most use cases, manual walking is ideal. If the goal is to automate with tooling, or be exhaustive, then the exponential problem returns. That said, this algorithm is still more efficient than existing algorithms in that case, as by the time we pay the exponentiation price, we will have already reduced the search space by running validation. Existing algorithms need to pay the exponentiation price in validation.
