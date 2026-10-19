# CONDA — H1–H3, falsification-first research plan

**Status (2026-10-18):** No hypothesis validated experimentally. A source-level mechanism is not the same as a measured bottleneck; an existing paper's observation is not our reproduction. Source reference anchor base: https://github.com/darae-lee/CONDA/blob/4117a4e38938aa0ff583c0c7c1beb5bb465e5b4b/ . Paper https://www.vldb.org/pvldb/vol19/p3357-lee.pdf .

## H1 — Repeated graph maintenance in clustered insertions

**Proposition:** Highly correlated inserts may contend for the same adjacent vertices, triggering repeated `InterInsert_with_expand` / re-pruning and additional edge churn that a careful, transactional, selective batch repair *might* avoid.

**Visible mechanism:** `src/index.cpp:131–153` parallelizes calls to `Insert_crng` with OpenMP; `156–306` computes candidates and prunes for *each* inserted vector, and `232–235` explicitly removes and replaces some edges. Reverse-link expansion is invoked at `305`; implementation resides in a binary-only archive. The graph store locks node adjacency individually: `include/store.h:79–137`. The paper's Algorithm 3 (p.3363, §4.3) details an MRNG re-pruning stage for overflow during reciprocal edge creation.

**Paper and prior art:** **ALREADY ACKNOWLEDGED** in CONDA §5.4.2 (p.3366); clustered batches with correlated similar data yield repeated re-pruning. Greator (PVLDB 19(3), DOI `10.14778/3778092.3778108`) directly proposes similarity-aware local connections to avoid redundant repair edges/re-pruning. Therefore a broad “batch duplicate work” pitch is not novel. The only theoretical differentiation is specific to CONDA's CRNG-occlusion map and concurrent reverse-edge expansion; no evidence yet.

**Evidence grade:** mechanism **plausible and author-acknowledged**, **not independently measured**; actual reverse-edge path **opaque**.

**Instrumentation needed (once full executable available):** around `InterInsert_with_expand`, record exact per-insert duration, number of target vertices re-pruned, re-prune invocation count per vertex, incoming-edge attempted/appended/deleted, edge revisits by batch, time waiting on graph locks, candidate depth and `occlusion_map` size. For a fixed batch set, report duplicate target-reprunes and edit oscillation ratio (`A→B→A` over timestamped epochs). Trace edge events on a single-thread controlled run first; parallel logs require per-thread ring buffers, monotonically ordered event IDs or per-node epochs, and measured logging overhead.

**Falsifier:** With identical vector sets at each batch boundary, matched `R/L/alpha/searchL`, and single-thread random vs clustered *within-batch* ordering, the median re-prunes per candidate and repeated edge-edit ratio **do not increase** for clustered order; or any increase is below run-to-run noise and consumes negligible update CPU time; or Greator-inspired selective repair matches the proposed improvement. Also reject if any speedup lowers recall@10, increases isolated/unreachable nodes, violates degree limits, or hides deferred maintenance outside the timing window.

**Experiment:** First pure observational profiling; later, if approved, compare untouched CONDA against an explicitly named duplicate-work suppressor, a no-op instrumented baseline, and an existing selective local-repair baseline. Paired five-seed trials at `OMP_NUM_THREADS=1`, then a small concurrency sweep after verifying sequential invariants. Pre-specify one-sided expected outcome (higher re-prunes in clustered order) and report per-seed confidence intervals. Complexity risk: shared mutable graphs make deferred/committed edge updates noncommutative; violating 2-hop monotonic paths or lost reverse reachability can appear as deceptively faster updates.

**Provisional novelty:** insufficient. **Do not optimize/pitch yet.**

## H2 — Redundant distance and candidate comparisons across similar inserts

**Proposition:** Correlated insertions may visit nearly identical neighborhoods and perform repeated candidate-to-candidate or target-to-neighbor distance computations. Reuse might reduce CPU work if inputs/graph state permit valid cached results.

**Visible mechanism:** `src/index.cpp:691–742` gathers graph neighbors and computes distance for visited new candidates; `249–263` evaluates connected pairs during 2-hop occlusion, and `265–293` implements optional 3-hop branch. `include/store.h:167–185` wraps metric kernels. `include/distance.h:20–22` defines `compare_count`, but at least two counter increment calls (`54`, `240`) are **commented out**; this is a logging capability limitation, **not measured zero comparisons**.

**Paper and prior art:** CONDA §4.2 already restricts candidate pair distances to graph-connected pairs. Existing localized repair and graph search prior art weakens the novelty of generic caching. There is **no verified citation proving that cross-insert pairwise caching of this exact CRNG operation is already implemented**; nor evidence that it saves meaningful work. Thus do not assert either novelty or prior-art identity.

**Evidence grade:** **speculative**. Dataset locality is not equal to identical operands or valid reuse; common candidate IDs are necessary but insufficient.

**Instrumentation:** count calls by `get_distance` overload and actual metric type, deduplicate unordered `(nodeID_a, nodeID_b)` pairs *within one well-defined graph epoch*, count unique/repeated computations and time, size the cache in bytes. Split search node–new-vector distances from graph node–node distances (the former cannot be reused merely because vectors are nearby). Quantify reuse distances by operation age/batch and graph mutations. Start by verifying/locally enabling a non-algorithm-changing counter only in an **isolated instrumented copy** after permission (not done now).

**Falsifier:** Pair-distance repeats are rare; lookup/hash/synchronization costs exceed saved metric evaluation; SIMD distance kernels are cheap relative to neighbor copying and graph locks; cache invalidation or staleness under slot reuse erases benefit; no repeatable improvement at fixed recall/connectivity, or exact same strategy exists in a direct prior publication.

**Experiment:** inspect repetitions under two-hop mode and `OMP_NUM_THREADS=1`, then compare an instrumentation-only control with a bounded exact-value cache keyed by `(vectorID, slot generation, metric)` only after allowing an isolated experimental algorithm implementation; never cache new-vector distances across different vector contents. Track p95/p99, peak RSS, hit rate and per-metric variance. Simultaneous graph mutations and re-used slots require generation-aware keys and concurrency correctness checks.

**Provisional novelty:** indeterminate but too weak for a research pitch. **Do not optimize/pitch yet.**

## H3 — Deletion consolidation scans or redundantly traverses CONDA's entire graph

**Proposition:** CONDA's delete path may perform avoidable graph scans or repeat too much localized repair.

**Crucial code distinction:** `src/index.cpp:322–387` contains an explicit full-graph `0..max_location` inspection. **This is `ConsolidateDelete` used by FV, not CONDA**. The CONDA test runner (`tests/test.cpp:299–319`) calls **binary-only** `MultiProcess_crng` before `ConsolidateDelete_conda` (`src/index.cpp:662–687`). The visible CONDA cleanup visits just the deleted IDs and does not scan all active graph nodes. Paper §4.4 proposes search-local repair and lazy cleanup specifically to eliminate full scans.

**Evidence grade:** the **full-scan formulation is contradicted** by observed benchmark dispatch; **localized repair expense unknown** due opaque `MultiProcess_crng` and `Process_crng`. Neither is proof of zero hidden scans; no code trace available for the compiled implementation.

**Instrumentation if reconsidered:** independently time `MultiProcess_crng` vs `ConsolidateDelete_conda`; record per-deleted-point greedy-search expansions, visited/in-neighbor candidates, re-prunes, distance evaluations, attempts to repair edges to already-deleted IDs, tombstone count, graph reachability/recall over long update horizons and total wall-clock including all deletion stages. Record number of active graph locations touched, not just function names.

**Falsifier:** `MultiProcess_crng` performs only bounded local work, maintains recall with no measurable scan proportion; or any alleged speedup shortens visited candidates and lowers true in-neighbor coverage, worsening stale edges. If measured runtime grows with `N` at fixed deletion count, first distinguish memory/NUMA and density effects from an actual `O(N)` scan.

**Scientific alternative grounded in author admission:** CONDA §7 notes that deletion greedy search may miss affected in-neighbors, leaving stale edges. Auditing coverage and connectivity after repeated deletions may be a worthwhile independent **reproduction** study, but the issue itself is already identified by the original researchers and repair logic is binary-only. **It is not a new H3 optimization proposal today.**

**Verdict on H3:** **reject original scan hypothesis**. Avoid pitching a false claim to the authors.

## Shared experimental quality gates

- Original algorithm unchanged for baseline; counter-only modifications only in a separate authorized clone/copy with reported overhead.
- Compare **the same canonical vectors and query set at each batch boundary**; randomize/sort *inside* each fixed batch. `OMP_NUM_THREADS=1` initially prevents OpenMP schedule noise from invalidating ordering treatment.
- Fixed `R`, `L`, `alpha`, metric, search width, batch size, queries, hardware, compiler flags. Repeat at least five deterministic seeds where resources permit; stratify by cluster tightness and overlap.
- Accuracy: exact brute-force recall@10 against currently active labels (using stored float32 values); both aggregate and per-cluster; do not report QPS without matching recall.
- Graph correctness: directed degree `≤R` at completed update boundaries, no active edge to invalid/unallocated IDs, indegree-zero count, reachable fraction from entry point, strongly connected components when meaningful, count of stale edges to deleted nodes (compare with expected semantics), and thread sanitizer on separate instrumented build for concurrency hypotheses.
- Time: total wall clock through all maintenance stages, per-operation p50/p95/p99, search QPS + tail latency, peak RSS, graph edge additions/removals. Report overhead of profiler itself; no cherry-picking.
- Reject if gain arises from silent deferred work, graph degree violation, lower recall, weaker reachability, incomplete update, or unequal active datasets.
- A benchmark must actually run for numeric claims. **No CONDA performance numbers were generated in this audit.**