# CONDA — Reproduction status and runnable synthetic fixtures

**Audit environment:** 2026-10-18 UTC, Linux x86_64, Git 2.47.3, CMake 3.31.6, GCC 14.2.0, Boost headers 1.83; system `ar` and `nm` available. A `yaml-cpp` development-package listing was absent in the package-query output; its availability must be checked on the target machine.

## 1. Actual execution status — do not misrepresent

| Task | Status | Evidence |
|---|---|---|
| Verify public HEAD and complete tracked-tree inventory | **Completed via read-only GitHub connector** | commit `4117a4e38938aa0ff583c0c7c1beb5bb465e5b4b`; see `EVIDENCE_LEDGER.md` |
| Read visible `src/index.cpp`, `include/store.h`, `include/index.h`, `tests/test.cpp`, build files | **Completed via GitHub connector** | exact line references in codebase report |
| Decode static archive container and symbol table | **Completed in prior research tool session**, binary not locally mounted | archive 674,438 bytes, 2 ELF objects, defined symbols; see ledger |
| Clone source into local working directory | **Failed** | `logs/remote_access.log`: DNS could not resolve `github.com` |
| Build untouched CONDA with `cmake && make` | **NOT RUN** | full source/archive not accessible in local container; `logs/build_attempt.log` |
| Run official benchmark or new CONDA profiler | **NOT RUN** | no runnable CONDA binary, no attempt to fabricate results |
| Make deterministic small synthetic inputs | **Completed** | `scripts/make_synthetic_workloads.py`; `logs/fixture_generation.log` |
| Validate controlled data identity at update boundaries | **Completed (Python fixtures only)** | four fixed 160-point batch-set equality assertions; not a graph algorithm test |
| Verify fixture regeneration byte-for-byte | **Completed** | `logs/fixture_rebuild_check.log` reported `True files: 7` |

### Exact blocked command and observable message

Attempt:

```bash
git ls-remote https://github.com/darae-lee/CONDA.git HEAD
```

Result:

```text
fatal: unable to access 'https://github.com/darae-lee/CONDA.git/': Could not resolve host: github.com
```

There was **no valid local source directory** against which to run CMake. A fake compiler command that is guaranteed to fail due missing source would add no scientific evidence. The precompiled ELF archive was inspected through a read-only GitHub response in the earlier tool session; it was **not** present as a local file on which `nm` could be run.

## 2. What was created and validated locally

Reproduction fixture generator at `scripts/make_synthetic_workloads.py` (Python standard library only):

- Fixed RNG seed **1701**, **640** stored float32 vectors, **8** dimensions, **4** distinguishable Gaussian clusters, **80** float32 queries.
- Four sets of **160** canonical vector IDs; **the same canonical set after each of the four completed batches** for both treatments.
- Inside each fixed batch, `random` shuffles the input order; `clustered` groups the *same* vectors by their cluster label. Consequently the treatment is **within-batch update order / temporal clustering**, not a different point population. Final identical data cannot guarantee identical graphs or equal intermediate states *during* a batch.
- `fixtures/{random,clustered}_base.fbin`, `fixtures/queries.fbin`: little-endian 32-bit `(count,dimension)` header followed by float32 rows, as required by `tests/test.cpp:23–46`.
- `fixtures/synthetic_runbook.yaml`: 8 steps (`insert` then `search` repeated four times), `max_pts=640`; matches `tests/test.cpp:69–95` YAML keys.
- `fixtures/{random,clustered}_groundtruth.json`: brute-force squared-L2 top-10 **row indices** computed on the exact quantized float32 input and active prefix at each search step. These are **reference labels**, not ANN recall results.
- `fixtures/fixture_manifest.json`: SHA-256 hashes, parameters, caveats.

Executed successfully:

```bash
python /mnt/data/CONDA_RESEARCH_AUDIT/scripts/make_synthetic_workloads.py
# status: fixture_verified; vector_set_checkpoints: 4; n:640; dim:8; queries:80
# note: No CONDA benchmark executed
```

Follow-up re-run hashed seven fixture files before and after regeneration and verified **byte-for-byte equality = True** (`logs/fixture_rebuild_check.log`). An early generator bug was corrected before this successful validation; no failed CONDA run occurred.

## 3. Strongest next reproducible *CONDA* test once source is transferred

Use a **separate local scratch clone**, pinned to the audited commit, with the distributed archive intact. This is a future recipe; **the following commands were not executed here**:

```bash
# Run on a machine with permitted Git access, not in the original upstream repository.
git clone https://github.com/darae-lee/CONDA.git CONDA-audit-scratch
cd CONDA-audit-scratch
git checkout 4117a4e38938aa0ff583c0c7c1beb5bb465e5b4b
git rev-parse HEAD
sha256sum lib/libconda_internals.a
ar t lib/libconda_internals.a
nm -C --defined-only lib/libconda_internals.a | grep -E 'InterInsert_with_expand|MultiProcess_crng|sync_prune|get_neighbors_lazy'
# On Debian/Ubuntu, install yaml-cpp, Boost, OpenMP first if missing.
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j2
```

The fixture directory can be copied into `CONDA-audit-scratch/fixtures/` (not pushed upstream). Then the untouched binary runner for the two input orderings:

```bash
export OMP_NUM_THREADS=1  # preserve within-batch experimental ordering; no parallel scheduling confound
for mode in random clustered; do
  mkdir -p "local_results/$mode"
  ./build/tests/test \
    fixtures/synthetic_runbook.yaml synthetic \
    "fixtures/${mode}_base.fbin" fixtures/queries.fbin \
    32 16 200 1.2 50 10 \
    "local_results/$mode" float CONDA L2 \
    > "local_results/$mode/run.log" 2>&1
done
```

This is an **exploratory small synthetic** configuration, **not the authors' 1M / 10M / 30M claimed reproduction**. Params shown are illustrative small-workload settings (`L=32, R=16, C=200, alpha=1.2, L_search=50, K=10`); benchmark table params are in `scripts/run_experiments.sh:13–14` and should be used for a paper-comparable study when datasets are available.

**Important:** real dynamic graph insertions may have hidden minimum-size or exceptional initial-state requirements. First demand clean execution and valid output shape; stop at faults. If the source archive is incompatible with the current toolchain, report the linker/compiler output verbatim instead of patching the upstream implementation. CPU supports and `-march=native` should be recorded.

## 4. Required outputs not available in current audit

Every cell below is **UNMEASURED**, not zero. A valid future result needs actual CONDA execution and fidelity checks.

| Requested metric | Current status | Minimal reproducible measurement |
|---|---|---|
| Total update wall time | UNMEASURED | benchmark's `BatchInsert execution time` and delete wall time, include all stages |
| Per-insert p95 / p99 | UNMEASURED | timed events inside insert around graph locks; record profiling overhead |
| Pair distance calls per insert | UNMEASURED | metric-specific count; existing counter increment in `distance.h` appears commented |
| Neighbor comparisons | UNMEASURED | branch and set-membership counts around CRNG and hidden prune routines |
| Graph edge adds / removes | UNMEASURED | counters in scratch `AddEdge/DeleteEdge/SetNeighbors`; count list differences and stale duplicates |
| Connectivity and reachability | UNMEASURED | graph adjacency snapshot, directed BFS from entry point, in-degree-zero, SCC size, degree bound |
| Recall@10 | UNMEASURED | compare `.res` row IDs per search step to corresponding `*_groundtruth.json`; average at fixed K=10 |
| Search throughput/p95/p99 | UNMEASURED | timed query batch and per-query measurements at matched recall |
| Peak memory | UNMEASURED | `/usr/bin/time -v` peak resident size; repeat runs |

**Caution:** The built-in runner primarily reports aggregate `BatchInsert`, deletion and search time at operation steps (`tests/test.cpp:252–323`), **not** per-insert tail or edge churn. Claiming its stock logs directly supply p99 per insertion would be false. Changing logging in a scratch clone requires a measured instrumentation-only baseline, permission and transparent diff.

## 5. Scientific correctness of the comparisons

- The matched-set invariant is true **at full batch boundaries only**. Search after each batch is comparable; intermediate per-insert states differ by design.
- `OMP_NUM_THREADS=1` is essential for initial ordering effects. For concurrent tests, use a deterministic release schedule or instead change the question to parallel contention rather than input-order effect.
- Query row content remains identical; tag-to-vector mapping differs by file order. Compare against the **matching mode's** groundtruth, not a shared vector-ID mapping.
- Before any performance claim, validate unmodified baseline's recall, path reachability, degree budget and edges to deleted nodes; repeated seeds and fixed hardware are essential.
- For proposed deferment, time deferred repair within the measurement boundary and verify correctness after each advertised update step. Improvements from incomplete maintenance are disqualified.

**Reproduction disposition:** the dataset/ground-truth **fixture** was built and validated. An actual CONDA performance/accuracy/graph reproduction is **blocked**. This is not a scientific success/failure result for the algorithm.