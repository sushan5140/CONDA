# CONDA — Evidence ledger, provenance and unperformed steps

This ledger distinguishes independent publisher/URL checks, read-only repository observation, source-based inference, offline fixture execution, and missing evidence. **Neither a future command in a report nor a paper's published result counts as an independently reproduced benchmark.**

## Evidence classes

- **P** = publisher/paper text: peer-reviewed author's *reported* results, not ours.
- **S** = directly reviewed public source file/repository metadata (read-only GitHub connector).
- **B** = precompiled archive structural/symbol inspection (not source semantics).
- **X** = actual local execution in working container, with log/path.
- **I** = theoretical inference drawn from source and labeled as such.
- **U** = unmeasured/unverified, must not enter empirical claims.

## Repository, paper and tooling provenance

| ID | Class | Source / path / exact location | Verified fact | Limits |
|---|---|---|---|---|
| E01 | P | https://www.vldb.org/pvldb/vol19/p3357-lee.pdf p.3357 lines 0–30, DOI `10.14778/3836663.3836694` | Lee + Kim, title, *PVLDB* 19(11):3357–3370 2026, affiliations printed as NAVER / KAIST | VLDB conference site lists both as KAIST; paper affiliation takes priority for bibliographic assertion |
| E02 | S | GitHub `branches/main` API `https://api.github.com/repos/darae-lee/CONDA/branches/main` | HEAD `4117a4e38938aa0ff583c0c7c1beb5bb465e5b4b`, authored 2026-09-14T14:21:36Z, tree `c483c2b2330cd64b41eea8c02c5f23621821aff3` | Not a local `git rev-parse`; connector response |
| E03 | S | `https://api.github.com/repos/darae-lee/CONDA/git/trees/HEAD?recursive=1` | complete visible tracked directory inventory includes source, three YAML runbooks, scripts, headers, `lib/libconda_internals.a`; no `conda_internals.cpp` | API returned `truncated:false`; absence describes *that tracked tree*, not authors' private source |
| E04 | S | `src/CMakeLists.txt:3–10` | project explicitly excludes `conda_internals.cpp` from glob and links archive | Link succeeded? **UNKNOWN**, never built locally |
| E05 | S | `src/index.cpp:131–153,156–306` | CONDA batching is OpenMP per insert; CRNG checks/edge rewires visible; calls binary-only reverse expansion | Hidden expansion time/branching **UNKNOWN** |
| E06 | S | `src/index.cpp:322–435` | full location scan belongs to `ConsolidateDelete`, `RefineDelete` baseline | Cannot call CONDA's delete logic |
| E07 | S | `tests/test.cpp:299–319` | `FV` → `ConsolidateDelete`; `IV` → `MultiProcess_ip`/`ConsolidateDelete_ip`; `CONDA` → `MultiProcess_crng`/`ConsolidateDelete_conda` | Opaque `MultiProcess_crng` cannot be source traced |
| E08 | S | `src/index.cpp:662–687` | visible CONDA physical consolidation loops over deleted IDs, no full node-range scan | Not proof hidden deletion stage lacks other expensive operations |
| E09 | S | `include/store.h:79–144,167–185,191–239` | adjacency copy/update locks; AddEdge O(1) amortized; DeleteEdge O(R) scan; distance compares use selected metric | Some graph operations are binary-only; concurrency linearizability unproved |
| E10 | S | `include/distance.h:20–22,54,240`, `include/store.h:69–75` | compare counter exists but selected `fetch_add` calls are commented out | Actual number of comparisons **NOT measured**, other kernels might differ |
| E11 | S | `tests/test.cpp:133–325` + `scripts/run_experiments.sh:13–14,87–90` | benchmark argv, YAML operations, mode dispatch, aggregate timing and stock parameter values | No per-insertion tail or graph-edge counters reported by stock runner |
| E12 | B | `lib/libconda_internals.a` Git blob `135d645e45828ad4d24efb1f6463784c525c51b7` | read-only retrieval decoded **674,438** bytes; UNIX `ar` magic; 2 ELF object members of ~313,784 and 331,032 bytes; named defined symbols for `InterInsert_with_expand`, `sync_prune*`, `get_neighbors_lazy`, `Process_crng`, `MultiProcess_crng`, `ConsolidateDelete_ip` in float/int8/uint8 instantiations | Custom ELF parser on connector base64 response; **NOT `nm` on locally cloned archive**, no decompilation or source semantics established |
| E13 | P | CONDA paper §4.2–4.4, Algorithms 2–4, pp.3361–3363 | CRNG 2-hop monotonic reachability, expansion for incoming links, search-local deletion repair | Design intentions are not performance measurements made here |
| E14 | P | CONDA paper §5.4.2, Table 3, p.3366 | Authors explicitly attribute repeated re-pruning under clustered arrivals; their table reports CONDA slower than Wolverine++ on clustered total updates | **Paper's numbers** not reproduced here |
| E15 | P | CONDA paper §7 p.3368 | Authors acknowledge deletion search can miss affected in-neighbors, leaving stale edges | Already identified by authors; cannot claim as novel discovery |
| E16 | P | Greator publisher https://www.vldb.org/pvldb/vol19/p495-yu.pdf ; DOI `10.14778/3778092.3778108` | selective local connections reduce needless edges/re-pruning for graph ANN updates | Disk-resident, not exact CONDA code; transferability unverified |
| E17 | P | Wolverine https://doi.org/10.14778/3734839.3734860 | 2-hop deletion path repair and refined candidate selection already studied (PVLDB 2025) | Different algorithm and objectives |
| E18 | P | CleanANN https://doi.org/10.1145/3816782.3819219 ; arXiv https://arxiv.org/abs/2507.19802 | concurrent fully dynamic ANN with workload-aware linking and lazy cleanup (SPAA 2026; arXiv variant called CleANN) | Distinguish from SPAA 2025 similarly named lock-free kNN tree work |
| E19 | P | IP-DiskANN https://arxiv.org/abs/2502.13826 | in-place deletion/insertion alternative to consolidation | **Preprint**; do not claim main-conference peer-reviewed source |
| E20 | X | `logs/remote_access.log` | local `git ls-remote` failed `Could not resolve host: github.com`; Git 2.47.3, CMake 3.31.6, GCC 14.2.0 | Local cloning/build **BLOCKED**. No auth/permission issue claimed |
| E21 | X | `logs/build_attempt.log` | records **no CONDA cmake/make attempt** due unavailable sources | Not a build failure of CONDA |
| E22 | X | `scripts/make_synthetic_workloads.py`, `fixtures/fixture_manifest.json`, `logs/fixture_generation.log` | 640 float32 vectors, 8 dims, 80 queries, four matched 160-point update sets; deterministic two-order fixtures and exact brute-force labels produced | **NOT an execution of CONDA** |
| E23 | X | `logs/fixture_rebuild_check.log` | 7 output files verified byte-for-byte stable under repeat generation | tests only fixture reproducibility |
| E24 | U | all candidate hypotheses | No measured CONDA insertion/deletion tail, distance counts, edge modifications, connectivity or recall | DO NOT infer quantitative performance gain |
| E25 | I | `src/index.cpp:211–295` plus `include/store.h` | theoretical visible CRNG two-hop complexity envelope `O(R C d)` plus sorting/set work, assuming R bounded, C candidates, d dimensions | Not an asymptotic proof of binary-only full procedure |

## External references verification status

**Peer-reviewed / confirmed metadata:**

- CONDA: https://www.vldb.org/pvldb/vol19/p3357-lee.pdf , doi:10.14778/3836663.3836694.
- Wolverine: https://doi.org/10.14778/3734839.3734860 , PVLDB 18(7):2268–2280 (2025).
- Greator: https://www.vldb.org/pvldb/vol19/p495-yu.pdf , PVLDB 19(3):495–508; published volume issue date 2025, VLDB 2026 program.
- CleanANN: https://doi.org/10.1145/3816782.3819219 , SPAA 2026 pp. 247–260, final title differs from arXiv (CleANN).
- SPFresh: https://doi.org/10.1145/3600006.3613166 , SOSP 2023.
- NSG: https://doi.org/10.14778/3303753.3303754 , PVLDB 2019.

**Preprints / not verified as peer-reviewed main-track proceedings:**

- FreshDiskANN https://arxiv.org/abs/2105.09613 ; IP-DiskANN https://arxiv.org/abs/2502.13826 ; IPGM https://arxiv.org/abs/2206.10839 .

## Commands actually executed

```bash
# Local environment inspection and read-only network test
command -v git cmake g++ nm ar
git --version
cmake --version
g++ --version
git ls-remote https://github.com/darae-lee/CONDA.git HEAD

# Fixture input generation and repeat-determinism check
python /mnt/data/CONDA_RESEARCH_AUDIT/scripts/make_synthetic_workloads.py
# then a short Python script to hash seven fixtures before/after regeneration
```

**Commands NOT executed:** `git clone`, `cmake -S . -B build` on CONDA, `cmake --build`, `ar t`/`nm` on a locally materialized `.a`, `./build/tests/test` (CONDA benchmark), profiling/perf, any source modifications/commits/pushes/emails.

## Research integrity notes

- Source/audit context from remote GitHub tools can be inspected despite working-container network being blocked; **different tool execution environments**, no contradiction.
- Paper Table 3 metrics are third-party published observations only. No values were copied into the audit as if observed locally.
- One early generator typing error was corrected prior to a successful final run; final script and logs are the reproducible state. This is not evidence of a CONDA defect.
- The strongest scientific conclusion is that **no hypothesis has cleared the novelty + executable-test + outcome gates**, rather than asserting a faster algorithm is impossible.