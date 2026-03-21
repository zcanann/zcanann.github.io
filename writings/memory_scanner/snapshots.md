# Snapshots & Filters

Most memory scanners burn a lot of effort on the wrong side of the problem. They obsess over the compare, and then quietly waste a ridiculous amount of time on memory layout, page boundaries, allocations, and result bookkeeping.

The snapshot/filter system in Squalr exists to attack that whole stack at once.

The short version is this:

- We load process memory into a format optimized for scanning.
- We keep that layout around.
- We scan the same snapshot without re-reading memory or fragmenting our reads.
- We store ranges of hits instead of inflating millions of individual results.
- We only materialize real scan results when the user actually asks to see them.

This is what lets the scanner be both very fast and very scalable. The compare kernels matter, obviously. But if the data layout is trash, the kernels do not save you.

## Snapshot Definition
A snapshot is often considered a view of process memory at a moment in time, but this definition misses important nuance.

A snapshot begins by querying all memory ranges of a target process, but the critical insight is to merge adjacent readable pages into larger **snapshot regions**. Finally, we read bytes into those merged regions.

So if the OS gives us:
- `0x1000..0x2000`
- `0x2000..0x3000`
- `0x3000..0x4000`

We do **not** treat that as three unrelated blobs. We merge it into one snapshot region:

- `0x1000..0x4000`

We still remember the original internal page boundaries, because reads can fail later. By storing these, we can implement tombstoning and recovery strategies for handling deallocations (ie if the `0x2000..0x3000` is deallocated later). But for scanning, the big win is obvious, namely that the scanner sees one contiguous byte array.

Without merging, every scan implementation has to constantly worry about:
- Running off the end of a page.
- Splitting filters at page boundaries.
- Special casing array scans.
- Special casing overlapping loads.
- Reconstructing larger logical windows from smaller physical pages.

That is all extremely difficult to manage, and merging avoids all of this pain. Another big win is that this makes the whole system more SIMD friendly, because vector kernels want to chew through contiguous memory, not bounce between tiny page-fragmented regions.

## Understanding Bottlenecks
The real bottleneck in scanning is usually value collection, not comparison.

Reading remote process memory is expensive. Scanning local bytes is cheap, especially when the bytes are:
- Contiguous
- Reused across scans
- SIMD friendly
- Represented through compressed filters instead of inflated result objects.

That is the whole point of this subsystem.

The snapshot system is what makes the rest of the scanner practical. It lets us pay the memory-read cost once, then attack the resulting byte arrays from many angles without rebuilding the universe each time.

## Red/Black Snapshot Buffers
Snapshots store both:
- `current_values`
- `previous_values`

This is what powers relative scans like changed, unchanged, increased, decreased, and delta-based comparisons.

Considering process memory I/O dominates the cost, we want to avoid massive allocations. Imagine scanning a 16GB game, and having to allocate 16GB of our own memory for each scan. If we only retain current/previous values, we can recycle allocations. So the region reader effectively does a red/black swap:
\[
\text{previous} \leftarrow \text{current}, \quad \text{current} \leftarrow \text{reused buffer}
\]

In code terms, the vectors are swapped and reused. The first couple of snapshots pay the allocation cost. After that, we mostly just recycle the arrays we already own.

Once the buffers exist, rescans are mostly paying for process memory reads, not allocations.

## Tombstones and Failed Reads
Real processes are messy. Pages disappear. Protections change. Reads fail.

When that happens inside a merged snapshot region, we do not want the entire region to become useless just because one interior page failed. So the system tracks page boundaries and tombstones failed boundaries.

This is important because merged regions are a scanning convenience, not a lie. Underneath, we still know where the original virtual pages were. If one of those pages becomes unreadable, we can track that fact without throwing away the whole merged region abstraction.

## Filters, Not Inflated Results
This is where the system starts getting clever.

A scan does **not** immediately produce a giant vector of concrete scan results. That would explode memory usage for no good reason.

Instead, a scan produces **filters**.

A snapshot filter is basically a window over a snapshot region that says, "the values that survived live in this address range under this data type and this alignment." It is much closer to a compressed structural description of hits than a literal list of results.

If a 0x2000-byte region is entirely zeros, and we scan `u8 == 0`, we do not need 8192 separate results. One filter can represent the whole thing.

That is the entire philosophy of this system:

- keep the snapshot big,
- keep the results compressed,
- only pay object inflation costs at the UI boundary.

## Anonymous Values
Another perk of this architecture is the concept of `AnonymousValueStrings`. Since scanning and storing filters are dirt cheap in our architecture, scanning for multiple data types simultaneously is extremely cheap. In fact, the primary bottleneck for scans is in reading memory. The process I/O bondary dominates the time costs.

From this, we build a system where we scan the same snapshot for an "anonymous" value, ie `0`, which can mean many different things depending on the interpretation:
- `i8`
- `i16`
- `i32`
- `i64`
- `u8`
- `u16`
- `u32`
- `u64`
- `f32`
- `f64`
- `...`

And we do not need to reread memory for each type. This is also why the data layout matters so much. If the snapshot representation were not SIMD friendly, the theoretical "many scans over one read" advantage would get eaten alive by internal overhead.

## Storing Filters as a `Vec<Vec<_>>`
At first glance, `Vec<Vec<SnapshotRegionFilter>>` looks less clean than flattening everything into one giant vector. As with all decisions we make, we do it because it is blazing fast.

The critical thing to keep in mind for scan results is not to squander the massive gains we get from SIMD and parallelization. This format avoids the problem by allowing the outer vector naturally maps to parallel work over snapshot regions or work chunks. The inner vectors let workers append local results without fighting over shared structures.

That means less synchronization, less reshuffling, and less cleanup work after the hot path is done. This avoids fighting over mutexes, and doing expensive serial copy or allocation operations at the end of a scan.


## Run-Length Encoding Is Doing Real Work
The filter system gets most of its compression from run-length encoding.

Suppose a compare kernel emits a stream of booleans over elements:
\[
[1,1,1,1,1,0,0,0,1,1]
\]

Instead of storing all ten booleans, we encode matching runs as ranges.

If a giant region fully matches, we encode a giant range.
If a giant region fully fails, we skip a giant range.
If the region is mixed, we only break where reality forces us to.

This pairs beautifully with SIMD. A wide compare often produces one of three outcomes:
- all false,
- all true,
- mixed.

The first two cases are absolute gold. They let the run-length encoder skip or accept whole SIMD-width chunks with almost no extra work.

That is why common values like `0`, `1`, or `0xFF` can scan absurdly fast. The key insight is that these common values dominate most process memory. In fact, the vast majority of process memory is simply `0`.

## Alignment Is More Powerful Than It Looks
Alignment is not just a user-facing scan option. It is part of the compression story. Suppose memory alternates like this:
\[
00, FF, 00, FF, 00, FF, \dots
\]

If you scan `u8 == 0` with alignment 1, you get fragmented hits everywhere. If you scan the same bytes with alignment 2, the scanner is allowed to treat every other byte as irrelevant. Suddenly the same logical results can collapse into a much cleaner filter layout.

This matters because alignment lets us preserve long contiguous windows of useful elements without physically splitting them into tiny fragments. In other words, alignment is not just skipping work. It is also preventing result fragmentation.

## Lazy Scan Result Materialization
Eventually the UI wants actual rows. Addresses. Typed values. Previous values. Display strings. Icons.

That is all expensive if we have millions of scan results. Naturally, the user does not need to see all of the millions of results at once, so we paginate. The retrieval path is intentionally pretty basic:
1. Walk snapshot regions until we find the region containing the requested result window.
2. Walk the filter collections inside that region until we find the filters covering the requested slice.
3. Materialize only those result rows.

This is fast enough because the number of snapshot regions is not huge, the filter collections already cache their result counts, and the expensive work only happens for the page the user is actually looking at.

A critical insight here is that we do not add excessive supporting data structures. We take advantage of binary search / linear seeking to query the nth page. Creating supporting data structures are a common pitfall that often cause cost blowup at the end of a scan, which we avoid entirely.
