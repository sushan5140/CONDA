# CONDA — Public codebase and binary boundary audit

**Audit date:** 2026-10-18 (UTC)  
**Repository:** https://github.com/darae-lee/CONDA  
**Pinned public `main` commit:** `4117a4e38938aa0ff583c0c7c1beb5bb465e5b4b` (GitHub branch API, committed 2026-09-14 14:21:36 UTC)  
**Exact Git tree at HEAD:** `c483c2b2330cd64b41eea8c02c5f23621821aff3` (commit's tree; the GitHub recursive `HEAD` tree API also returned SHA `4117a4...` as a lookup key).  
**Evidence basis:** repository metadata, recursive public tree, selected entire source files and line-ranged file reads through read-only GitHub integration; binary archive fetched and ELF symbol tables decoded in the research session. Local clone FAILED due network DNS; do not misread this as a successful full local source audit. No original files changed.  
**Source line URL pattern:** `https://github.com/darae-lee/CONDA/blob/4117a4e38938aa0ff583c0c7c1beb5bb465e5b4b/<path>#L<first>-L<last>`.

## 1. What is CONDA?

CONDA is an in-memory directed proximity graph for approximate nearest-neighbor search that handles insertions, deletions and queries. Its `CRNG` neighbor-selection rule considers graph-connected short paths rather than solely comparing distances. Insertion tries to preserve incoming reachability to new nodes; deletion uses search-local repair and lazy physical cleanup instead of making every deletion trigger a graph-wide in-neighbor scan. The paper introduces CRNG, in-neighbor candidate expansion and scan-free deletion as key contributions (paper §4.2–4.4, Algorithms 2–4).

Bibliography independently confirmed in publisher paper: Darae Lee (**NAVER Corporation** at publication) and Min-Soo Kim (**KAIST**), *PVLDB* **19(11):3357–3370** (2026), DOI **10.14778/3836663.3836694**. The VLDB conference program lists both as KAIST; this does not override the affiliations printed in the publication.

Publisher PDF: https://www.vldb.org/pvldb/vol19/p3357-lee.pdf

## 2. Repository architecture and build

- `include/index.h:19–171`: public `efanna2e::Index<T>` API; declares `Insert_crng`, `BatchInsert_crng`, `InterInsert_with_expand`, `sync_prune` overloads, `sync_prune_crng`, `get_neighbors_lazy`, `MultiProcess_crng`, `Process_crng`, `ConsolidateDelete_conda`, `ConsolidateDelete`, and related baseline paths. Public declarations do **not** imply public source definitions.
- `src/index.cpp:30–748`: **visible** construction, insertion dispatch, `Insert_crng` (156–306), mark deletion (309–319), baseline `ConsolidateDelete` and `RefineDelete` (322–435), searches, `ConsolidateDelete_conda` (662–687), and non-lazy `get_neighbors` (691–742).
- `include/store.h:18–~274`: **visible inline** adjacency, locks, distance wrappers, deletion and recycled-location helpers.
- `include/distance.h`: **visible** distance kernel interface and SIMD implementation; counter exists but increment calls at lines 54 and 240 are **commented out**.
- `src/dynamic_index.cpp`: thin tag-map accessors; `include/dynamic_index.h` contains mappings.
- `tests/test.cpp:133–340`: YAML workload engine and dispatch for `FV`/`IV`/`CONDA`.
- `scripts/run_experiments.sh:13–115`: fixed workload params `BUILD_L=64`, `R=32`, `C=750`, `ALPHA=1.2`, `SEARCH_L=100`, `SEARCH_K=10`; uses built `build/tests/test`.
- `runbooks/{clustered_runbook,sliding_window_runbook,expiration_runbook}.yaml`: pre-written chronological operations; clustered profiles large (10M/30M).
- `CMakeLists.txt:1–21`: GCC C++17, OpenMP, `-O3 -march=native`; `tests/CMakeLists.txt:3–9` depends on `yaml-cpp`.
- **Critical:** `src/CMakeLists.txt:3–10` compiles all `.cpp` except any `conda_internals.cpp`, then links `lib/libconda_internals.a`. The recursive tracked file tree did **not** expose `conda_internals.cpp` source; the `.a` is a tracked binary blob (Git blob `135d645e45828ad4d24efb1f6463784c525c51b7`, **674,438 bytes**).

## 3. Exact call paths

### 3.1 Insertion (`CONDA`)

`tests/test.cpp:270–296` → `Index::BatchInsert_crng` (`src/index.cpp:131–153`) → OpenMP `Insert_crng` (`156–306`):

1. Reserve location and write tag mappings under `tag_lock_` `std::unique_lock<std::shared_mutex>` (`163–169`); copy vector into data store (`171`). First node sets `ep_` (`173–175`).
2. Select `get_neighbors_lazy` when `is_delete` (`182–188`), else non-lazy `get_neighbors` (`189–196`). **The lazy routine is binary-only**. Non-lazy search visibly starts from the entry point, visits neighbors, evaluates distances and uses a bounded priority queue (`691–742`).
3. Sort `expanded_nodes` (`198`); allocate occlusion/rewiring tracking arrays (`203–206`).
4. CRNG loop (`211–295`) decides retention via two-hop reachability and distance tests. In particular `GetNeighbors(p.id)` and hash set membership (`249–263`) only calculate `get_distance(p.id,q.id)` when the pair is connected (`257–260`). Optional `hop_count >= 3` traverses second-order neighbors (`265–293`); benchmark runner sets 2 (`tests/test.cpp:220`).
5. When a previous path is non-monotonic, invoke `DeleteEdge(result_loc,p.id)` and `AddEdge(result_loc,location)` (`232–235`). A visible actual graph mutation, **not** evidence it is redundant.
6. Materialize selected out-neighbors by `SetNeighbors(location, neighbors)` (`297–303`), then **binary-only** `InterInsert_with_expand(location,vector,parameters,expanded_nodes,pruned_id,maxid)` (`305`). Paper §4.3, Algorithm 3 describes reverse-edge candidate expansion and MRNG re-pruning, but exact binary control flow/counters remain unverified.

`BatchInsert_crng` inserts the initial point serially if the deleted set is empty, then processes remaining points under `#pragma omp parallel for schedule(dynamic,16)` (`137–151`). A batch here does **not** inherently consolidate repeated work; it is parallel invocation of single inserts.

### 3.2 Normal insertion and baselines

`Index::BatchInsert` (`69–81`) → `Insert` (`84–128`) → `get_neighbors` → **binary-only** `sync_prune` and **binary-only** `InterInsert`. This is the common `FV` and `IV` insert dispatch in `tests/test.cpp:288–292`, not the distinctive CRNG loop. Do not attribute its exact pruning operations to CONDA's specialized insertion.

### 3.3 Deletion — distinguish dispatchers

`tests/test.cpp:299–324` calls `MarkDelete` once per tag (`src/index.cpp:309–319`), then switches by algorithm:

| Mode | Actual benchmark path | Source visibility |
|---|---|---|
| `FV` | `ConsolidateDelete(parameters)` | `src/index.cpp:322–387` **visible**; scans every location `0..max_location` (`340–360`), collects affected nodes and calls `RefineDelete` (`391–435`) |
| `IV` | `MultiProcess_ip`, then `ConsolidateDelete_ip` | **binary-only** implementation; declarations visible in `include/index.h:81–90` |
| `CONDA` | `MultiProcess_crng`, then `ConsolidateDelete_conda` | first stage **binary-only**; second **visible** `src/index.cpp:662–687` |

**Critical counterexample to H3:** `ConsolidateDelete_conda` iterates **deleted IDs**, `ClearNeighbors`, fills vector slots with the maximum representable scalar, returns them to the empty-location set, updates size/entry point, and clears tombstones. It contains **no explicit `0..N` scan** (`662–687`). Thus the `ConsolidateDelete` graph-wide loop `340–360` cannot legitimately be called CONDA's deletion bottleneck. It is possible the opaque `MultiProcess_crng` contains expensive local traversal, but actual cost requires instrumentation, not guessing.

`MarkDelete` mutates tag-location maps and deleted set without the visible `tag_lock_`; it is used serially in the benchmark runner. No race can be diagnosed from that alone; separate concurrent tests would be required.

### 3.4 Adjacency and synchronization

`include/store.h`:

- `SetNeighbors`: `79–81`, locks one vertex exclusively, moves a new vector; **O(R)** to allocate/copy its input upstream; O(1) move and locking once constructed.
- `ClearNeighbors`: `111–114`, exclusive lock, clears list, **O(R)** worst case for resource processing.
- `AddEdge`: `118–121`, locks source vertex, `push_back`; amortized O(1) excluding capacity expansion. No duplicate check is visible in this helper.
- `DeleteEdge`: `123–131`, locks source vertex, `std::remove` + `erase` => **O(R)**.
- `GetNeighbors`: `133–137`, acquires shared lock and **returns a vector copy** => **O(deg(v)) ≤ O(R)** copy per call.
- `ForEachNeighbor`: `139–144`, visits under a shared lock, avoiding vector copy but holding the lock longer.
- `get_distance`: `167–185`, obtains vectors and delegates to metric-specific `distance_->compare`; normally **O(d)** for scalar pairwise metric, optimized SIMD can change constant.
- `delete_lock_` and `empty_lock_` protect some set operations (`191–239`); `tag_lock_` at `include/index.h:165–166` protects allocation and tag writes in the insertion method; graph locks are per-node.
- `get_deleted_set`/`copy_deleted_set` (`201–209`) return **references** while using a short-lived shared lock. If the set is mutated concurrently after the method returns, lifetime/synchronization merits a separate test; the current audit has **not** established such a race.
- `reserve_location` (`237–245`) is called under the insertion tag lock; no unsynchronized corruption was demonstrated.

**Lock ordering / safety caution:** `InterInsert_with_expand` is opaque; cannot verify whether multiple node locks, re-pruning, dynamic search, or readers interleave safely. Paper concurrency graphs cannot be taken as proof of linearizability.

## 4. Approximate computational costs (not benchmark measurements)

Let `d` be aligned vector dimensions, `R` graph degree bound, `C` expanded candidates for an inserted vector, `L` search-queue width, `D` tombstoned nodes, `N` occupied location range, `A` locally affected deletion nodes. The listed costs use worst-case straightforward operation counting for **visible portions only**:

| Operation | Visible cost model | Important qualification |
|---|---|---|
| `get_neighbors` (non-lazy) | visited-node traversal and `O(R)` neighbor copies per expansion, distance comparisons `O(d)` each; up to `O(C R d)` naive upper envelope | Actual `C` depends on queue behavior and reachable graph; `get_neighbors_lazy` opaque |
| CRNG two-hop loop (`Insert_crng:211–264`) | sorting `O(C log C)`, up to `O(R C)` membership tests, `O(R^2)` neighbor-set construction, and at most `O(R C)` pair distances `O(d)` | Do not claim all `R*C` distances actually executed: CRNG selectively measures connected pairs |
| CRNG 3-hop option (`265–293`) | adds up to `O(R^2)` reachable-neighbor processing per selected node, plus pair membership | Benchmark uses 2-hop mode |
| `DeleteEdge` (`store.h:123–131`) | `O(R)` scan/removal per call | The number of calls is **unknown** without counters |
| `ConsolidateDelete` for `FV` (`index.cpp:322–387`) | full `O(NR)` neighbor inspection before per-affected-node repair | **Not CONDA** |
| `ConsolidateDelete_conda` (`662–687`) | visible cleanup `O(D(R+d))` plus set and scheduling overhead | `MultiProcess_crng` hidden; full deletion total **unknown** |
| `InterInsert_with_expand`, `sync_prune*`, `Process_crng`, `MultiProcess_crng` | **cannot derive reliable internal counts or asymptotic details** | Present as compiled code only |

These symbolic envelopes are **not** observed timings, and do not bound the complete algorithm where implementations are opaque.

## 5. Static archive verification and limits

Read-only GitHub fetched `lib/libconda_internals.a` with base64 content (decoded in an isolated JavaScript audit). Observed UNIX `ar` magic `!<arch>\n`, **674,438** decoded bytes, archive member records `/`, `//`, `/0` (~313,784 bytes), `/19` (~331,032 bytes); **two ELF object members**. Direct symbol-table parser found named **defined** C++ instantiations for:

- `Index<T>::InterInsert_with_expand`
- `Index<T>::sync_prune` overloads; `sync_prune_crng`
- `Index<T>::get_neighbors_lazy`
- `Index<T>::InterInsert`
- `Index<T>::Process_crng` and `MultiProcess_crng`
- `Index<T>::Process_ip`, `MultiProcess_ip`, `ConsolidateDelete_ip`

Found symbol names for `T=float`, `T=int8_t`, and `T=uint8_t` and OpenMP outlined function names, consistent with the benchmark modes. Both ELF objects expose overlapping template-name listings; the audit did **not** perform authoritative link-time resolution or disassembly. Symbol presence proves *compiled implementations exist in the archive*, not their detailed semantics. **Do not treat symbols as source.** System `nm`/`ar` are installed but could not be run on an actual locally mounted archive because cloning/download of repository bytes was blocked. A follow-up local check should use `ar t`, `nm -C --defined-only`, `readelf -Ws`, and a link map after an authorized offline copy is available.

## 6. Quality threats / follow-ups

1. Graph locks provide individual adjacency-list synchronization, but do not automatically imply transaction-level atomicity of multi-edge rewiring.
2. Existing compare-count instrumentation appears inactive: `include/distance.h:54,240` shows commented `compare_count.fetch_add`. `get_compare_call()` exists in `include/store.h:69–74`, so blindly publishing compare counts from it would be invalid without checking all metric implementations.
3. The `CONDA` benchmark runner sets 2-hop pruning (`tests/test.cpp:220`), even though user-adjustable 3-hop branch exists. Do not claim the paper evaluated arbitrary hops.
4. `BatchInsert_crng` performs single-item insertions concurrently; a hypothetical batch coalescer would **change the algorithm** and its concurrent semantics, not simply tune an existing batch switch.
5. Deletion's heavyweight first-stage topology work is in `MultiProcess_crng` and requires source access or timing/sampling around calls to characterize.

**Summary:** substantial visible code reviewed and archive symbols inspected, but **complete implementation not independently source-auditable**, and **no CONDA build** has been executed. See `CONDA_REPRODUCTION.md` and `EVIDENCE_LEDGER.md`.