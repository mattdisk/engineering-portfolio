# High-Performance Algorithms

A collection of C, CUDA, OpenMP, and MPI implementations of classic parallel and memory-efficient algorithms. They cover three sources of cost in performance-oriented code: **computation** (shared-memory and GPU parallel list ranking), **communication** (MPI collectives built only from point-to-point messages, plus a distributed sort built around two MPI collective calls), and **memory transfers** (a cache-oblivious lazy funnel sort). Each implementation is the core routine behind a fixed function interface. The focus is on algorithmic choices: how work is decomposed, how processes exchange data, and how data moves through the memory hierarchy.

## Implementations

| Area | Implementations | Technologies | Problem | Key technical idea |
|---|---|---|---|---|
| **List ranking** | Wyllie pointer jumping; Helman–JáJá (CPU); Helman–JáJá (GPU) | C, OpenMP, CUDA, cuRAND | Compute every node's distance from the head of a linked list stored as a successor array | Fine-grained pointer jumping vs. coarse-grained random sublist decomposition; GPU version encodes sublist boundaries in place for O(1) lookup |
| **MPI collectives** | Reduce, Scatter, Gather, Allgather, Broadcast (tree), Broadcast (scatter + allgather) | C, MPI point-to-point only | Implement standard collectives without calling MPI's collective routines | Binomial / minimum-spanning-tree patterns for latency, a ring for bandwidth, and their combination for long-message broadcast |
| **Distributed sorting** | TeraSort-style distributed sort | C, MPI, OpenMP | Globally sort fixed-size key/value records spread across processes | Histogram-based splitter selection with a sweep-line merge; two MPI collective calls; key-plus-pointer LSD radix sort |
| **Cache-oblivious sorting** | Lazy funnel sort | C | Sort an array while minimizing cache misses without knowing cache or line size | Recursive K-funnel in one contiguous van Emde Boas–style layout, exact memoized space accounting, reduced middle buffers |

## Source Access

The implementation source is maintained in a private repository and is available to reviewers upon request.

## Algorithmic Design and Trade-offs

### List ranking: two parallel strategies for a sequential-looking problem

List ranking, the linked-list analogue of prefix sum, is hard to parallelize because list traversal is inherently sequential and its memory accesses are irregular.

**Wyllie (OpenMP).** Each node starts with weight 1 (the head with 0). In each of ⌈log₂ n⌉ rounds, every node adds its accumulated rank into its successor, then jumps its pointer to the successor's successor. Rank and pointer arrays are double-buffered and swapped between rounds, so each round reads only the previous round's state. This reproduces synchronous PRAM semantics without fine-grained locking. The cost is O(n log n) work and a barrier per round, in exchange for O(log n) depth and uniform, embarrassingly parallel loops.

**Helman–JáJá (OpenMP).** Choose *s* random sublist heads, always including the true head. Traverse the sublists in parallel to compute local ranks, prefix-sum the *s* sublist offsets sequentially, then make a second parallel pass that adds each sublist's offset. In the standard formulation the traversals take O(n) total work and the sequential step touches only *s* elements, which reduces both traversal work and synchronization compared with Wyllie. Here *s* = ⌈p · log₂ n⌉ for *p* processors: random splitters produce uneven sublists, and many sublists per thread smooth out the imbalance. This implementation finds a successor sublist's index with a linear scan of the head list when a traversal reaches a candidate boundary, which adds sublist-management overhead on top of the traversal work.

**Helman–JáJá (CUDA).** The GPU version restructures the algorithm rather than porting it:

- **In-place boundary encoding.** When a node is chosen as a sublist head, its entry in the device successor array is overwritten with −(i + 2), identifying sublist *i*, and the original successor is saved separately. A traversing thread detects the boundary and finds the next sublist's index in O(1), with no head-list search.
- **Count first, write once.** The first kernel only counts sublist lengths. After the offset prefix sum, the second kernel writes each node's final rank directly, so every rank is written exactly once.
- **Intermediate stages stay on the GPU.** Splitter selection (using cuRAND) and the O(s) prefix sum run as single-thread kernels. Only the successor array is copied in and only the rank array is copied out.
- **Granularity.** *s* = ⌈log₂ n⌉ × 128, capped at n/2 for small inputs. Each traversal is a chain of dependent, uncoalesced loads, so the GPU needs far more sublists than a CPU to hide memory latency through concurrency.

### MPI collectives: latency-bound vs. bandwidth-bound patterns

All six collectives use only `MPI_Send`, `MPI_Recv`, `MPI_Isend`, and `MPI_Waitall`. Costs below use the standard α–β model (α = per-message latency, β = per-element transfer time, *p* processes, *n* elements):

| Collective | Pattern | Cost (α–β model) |
|---|---|---|
| Reduce (sum) | Binomial tree: in round *k*, rank *r* pairs with *r* XOR 2ᵏ; senders drop out after sending their partial sum | ⌈log p⌉ (α + nβ), plus local additions |
| Broadcast (tree) | Minimum-spanning-tree (MST) recursive halving: each range's root sends the full message to the midpoint, then both halves recurse | ⌈log p⌉ (α + nβ) |
| Scatter | MST recursive halving, forwarding only the upper half's blocks, so message size halves per level; leaves receive directly into the output buffer | ⌈log p⌉ α + ((p−1)/p) nβ |
| Gather | Mirror of scatter: subtrees gather first, then forward their contiguous block upward | ⌈log p⌉ α + ((p−1)/p) nβ |
| Allgather | Bucket (ring): in each of p−1 steps, every rank forwards its most recently received block to its neighbor | (p−1) α + ((p−1)/p) nβ |
| Broadcast (scatter + allgather) | MST scatter of n/p-sized pieces, then a ring allgather | (⌈log p⌉ + p − 1) α + 2((p−1)/p) nβ |

The two broadcasts show the central trade-off. The tree broadcast minimizes message count but sends the whole payload log p times, which suits short messages. The scatter-allgather broadcast sends more messages but moves only about 2n data per process regardless of *p*, which suits long messages. Its ring phase is opened into a chain: the last rank never sends and the root never receives, since the root already holds the full buffer.

Implementation-specific choices:

- **Message-size–dependent sends.** In the tree broadcast, scatter, gather, and allgather, sends below 256 elements are blocking. Larger sends are posted with `MPI_Isend` and completed with one `MPI_Waitall`, so large transfers overlap with the rest of the schedule.
- **Contiguous block ranges.** Splitting the rank range into contiguous halves means every tree edge carries a contiguous buffer slice, so no packing is needed.
- **Simplifying assumptions**, enforced by `assert`: `MPI_INT` data, root rank 0, and `MPI_SUM` for reduce.

### Distributed sorting: histogram partitioning with two collective calls

The TeraSort-style sort operates on fixed-size records (10-byte key, 88-byte value) spread across MPI processes. Its design aims to send each record across the network at most once and to coordinate all ranks through two MPI collective calls: `MPI_Allgather` and `MPI_Alltoallv`.

- **Histograms instead of a local pre-sort.** Sample sort usually sorts each partition before choosing samples. Here, the first four key bytes are mapped to a float with base-256 positional weights, which preserves byte-lexicographic order up to floating-point resolution. Two linear OpenMP passes then compute the local min/max and a 7p-bin equal-width histogram, replacing an O(n log n) sort. The histograms are shared with `MPI_Allgather`.
- **Sweep-line merge with identical splitters on every rank.** The gathered bins come from different processes with different, overlapping ranges. Each bin is treated as a uniform-density segment with +d and −d events at its edges. Sorting and sweeping the events yields the merged distribution in O(K log K) for K bins, and the p−1 splitters are placed at equal-count quantiles by interpolating within segments. Every rank computes the same splitters independently, so splitter selection needs no further communication.
- **Receive buffers sized without probing.** Each rank estimates its incoming data volume from the histograms it already holds, so it can allocate the receive buffer before the exchange without `MPI_Probe` or a separate count exchange.
- **Metadata carried with the data.** A single `MPI_Alltoallv` performs the redistribution. Each outgoing segment begins with the sender's full per-destination count vector, from which every receiver derives the global output counts and displacements without another collective. A rank's own bucket is written directly into its receive buffer instead of being sent.
- **Compact local sort.** Since records are 98 bytes, the final sort runs on (key, pointer) entries: a stable LSD radix sort over all 10 key bytes, 256 buckets per pass. Each record is then copied once into its sorted position with a parallel `memcpy` loop.

`MPI_Alltoallv` is used deliberately as a standard library collective, since all-to-all personalized exchange is where vendor implementations can exploit topology and protocol details. The hand-built collectives above show how such algorithms are constructed from point-to-point primitives; this sort relies on the library routine where that is the practical choice.

### Cache-oblivious sorting: lazy funnel sort

Funnel sort achieves the optimal cache-miss bound for comparison sorting, O((N/B) log_{M/B}(N/B)), without knowing the cache size M or line size B (under the tall-cache assumption). The implementation follows the lazy variant from the cited references.

- **Recursive structure.** The input is split into K ≈ N^{1/4} runs of length N^{3/4}, each sorted recursively and merged with a K-funnel. Below 1,000 elements it falls back to quicksort. Two arrays alternate as source and destination across recursion levels, so results are never copied back.
- **Contiguous recursive layout.** A K-funnel is a top funnel over ⌈√K⌉ inputs, ⌈√K⌉ middle buffers, and ⌈√K⌉ bottom funnels of √K inputs each, recursing down to binary mergers. The entire funnel, including every `Merger` node, `Buffer` header, and buffer contents, lives in one allocation in van Emde Boas–style order: the top funnel first, then each middle buffer followed by the bottom funnel that fills it. Any sub-funnel small enough to fit in cache occupies a contiguous region.
- **Exact space accounting.** Each K-funnel's size is computed by a memoized recursion that includes the `Merger` and `Buffer` overhead, not just element storage. A smaller funnel lets more levels fit in cache, whereas an asymptotic estimate would over-allocate.
- **Smaller middle buffers.** Textbook descriptions often size middle buffers at about 2K^{3/2}, which supports the *eager* variant's refill-at-half-empty policy. The *lazy* variant refills a buffer only when it is empty, so buffers here hold ⌈√K⌉³ / 5 elements. The exponent and divisor are tuning parameters.
- **Merge mechanics.** Buffers are circular, and a merger pulls from its children only when an input runs empty. Once one input remains, elements move in contiguous `memcpy` chunks, handling wrap-around, rather than one comparison at a time.

The central tension is between **cache misses and wall-clock time**: tuning buffer sizes or the base-case threshold can lower the cache-miss ratio while increasing running time, because funnel bookkeeping and branch-heavy merging add instruction overhead that a tuned quicksort avoids.

### How the pieces relate

- **Computation:** Wyllie vs. Helman–JáJá contrasts a work-inefficient, highly synchronous algorithm with a coarse-grained, work-reducing one; the CPU and GPU Helman–JáJá versions show one algorithm restructured for a different execution model.
- **Communication:** tree vs. ring collectives expose the latency-versus-bandwidth trade-off, and the scatter-allgather broadcast combines both.
- **Memory transfers:** the distributed sort and funnel sort both minimize data movement, but at different levels. One limits network traffic and coordination between processes; the other limits cache–main-memory transfers within one process. They target different bottlenecks rather than competing.

## Implementation Scope

Each file implements one entry point against a fixed interface (for example `parallelListRanks`, `GT_Bcast`, `terasort`, `funnelSort`). Shared headers (record formats, comparison and utility routines), driver programs, data generators, test infrastructure, and build scripts are outside this collection. No benchmark results are included, and the descriptions above are not performance measurements.

## Private Source Organization

The private implementation repository is organized as follows:

```text
high-performance-algorithms/
├── list-ranking/
│   ├── listrank_wyllie.c       # Wyllie pointer jumping (OpenMP)
│   ├── listrank_hj.c           # Helman–JáJá (OpenMP)
│   └── cuda_hj.cu              # Helman–JáJá (CUDA)
├── mpi-collectives/
│   ├── reduce_tree.c           # binomial-tree reduce
│   ├── scatter_tree.c          # MST scatter
│   ├── gather_tree.c           # MST gather
│   ├── allgather_bucket.c      # ring allgather
│   ├── broadcast_tree.c        # MST broadcast
│   └── broadcast_sag.c         # scatter + allgather broadcast
├── distributed-sorting/
│   └── terasort.c              # histogram-splitter distributed sort (MPI + OpenMP)
└── cache-oblivious-sorting/
    └── funnelSort.c            # lazy funnel sort
```

## References

- J. C. Wyllie, [*The Complexity of Parallel Computations*](https://ecommons.cornell.edu/items/34f900d5-6e40-4ba4-a8e0-d571d0bce314), PhD thesis, Cornell University, 1979.
- D. R. Helman and J. JáJá, "Designing Practical Efficient Algorithms for Symmetric Multiprocessors," *ALENEX 1999*, LNCS 1619. [doi:10.1007/3-540-48518-X_3](https://doi.org/10.1007/3-540-48518-X_3)
- E. Chan, M. Heimlich, A. Purkayastha, and R. van de Geijn, "Collective Communication: Theory, Practice, and Experience," *Concurrency and Computation: Practice and Experience*, 2007. [doi:10.1002/cpe.1206](https://doi.org/10.1002/cpe.1206)
- O. O'Malley, ["TeraByte Sort on Apache Hadoop"](https://sortbenchmark.org/YahooHadoop.pdf), 2008; [Sort Benchmark](https://sortbenchmark.org/). Background on the TeraSort record format and benchmark; the implementation here is an independent MPI design.
- E. D. Demaine, ["Cache-Oblivious Algorithms and Data Structures"](https://erikdemaine.org/papers/BRICS2002/), lecture notes, EEF Summer School on Massive Data Sets, BRICS, 2002.
- H. Prokop, ["Cache-Oblivious Algorithms"](https://dspace.mit.edu/handle/1721.1/80568), MS thesis, MIT, 1999.
- F. Rønn, ["Cache-Oblivious Searching and Sorting"](https://hjemmesider.diku.dk/~jyrki/PE-lab/Frederik/thesis.pdf), MS thesis, University of Copenhagen, 2003.
- K. Vinther, ["Engineering Cache-Oblivious Sorting Algorithms"](https://cs.au.dk/~gerth/advising/thesis/kristoffer-vinther.pdf), MS thesis, University of Aarhus, 2003.

---

*This page reproduces the project's README. The source files and supporting documents it mentions are kept in a private source archive and are available on request.*
