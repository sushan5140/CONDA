# CONDA — Independent decision on prospective undergraduate supervision

**Decision date:** 2026-10-18  
**Purpose:** Whether **two prospective undergraduate collaborators** should contact Darae Lee or Professor Min-Soo Kim with a purported novel CONDA optimization proposal.  
**Decision:** **NO-GO for a research-contribution pitch at present.** This is **not** a judgment of the authors' research quality; CONDA is substantial published work. It is a judgment that we have not identified and validated a new, defensible *additional contribution*.

## 1. Scoring protocol (strict)

Maximum **100 points**: Novelty/prior-art gap **30**; Experimental evidence **25**; Scientific importance **20**; Implementation feasibility **15**; Alignment with CONDA authors **10**. **GO requires ≥74 AND independent testability AND no adequately established same central contribution.** Absent accessible internal C++ implementations and without a viable alternative test, a high speculative importance score cannot offset ineligibility. No candidate earns passing status.

| Candidate | Novelty /30 | Evidence /25 | Importance /20 | Feasibility /15 | Fit /10 | Total /100 | Rationale |
|---|---:|---:|---:|---:|---:|---:|---|
| H1: selectively suppress repeated clustered-update reverse-link repairs | 8 | 4 | 14 | 6 | 9 | **41** | The *problem* is explicitly acknowledged by CONDA §5.4.2 and addressed in an adjacent setting by peer-reviewed Greator; quantitative mechanism unmeasured; core re-pruning in compiled archive. |
| H2: reuse distances or candidate evaluation across similar inserts | 7 | 1 | 12 | 6 | 8 | **34** | Specific cache novelty not disproved but **not established**. Repetition and hit rate unknown; cache coherency requires slot-generation and concurrency guarantees; CRNG already selectively computes distances. |
| H3: eliminate full graph scan in CONDA deletion | 0 | 0 | 6 | 0 | 8 | **14** | The visible scan belongs to FV, not CONDA. CONDA's tested deletion dispatch is local repair plus lazy cleanup. Original premise is invalid. |

Scores are **reviewer judgments**, not benchmark results or precise scientific measurements. They are conservative eligibility scores under the stated evidence constraints; numerical differences of a few points cannot affect the decision.

## 2. Findings that disqualify a GO pitch

1. **Attribution error avoided:** do not confuse `ConsolidateDelete` (`src/index.cpp:322–387`, FV) with `ConsolidateDelete_conda` (`662–687`, CONDA). There is no visible full-scan deletion bottleneck to remove from CONDA.
2. **Author acknowledgement:** clustered correlated updates and repeated re-pruning are identified in the paper's §5.4.2, not a newly discovered failure mode. It takes algorithmic novelty and empirical evidence beyond restating that fact.
3. **Direct prior-art pressure:** Greator (PVLDB 19(3), DOI `10.14778/3778092.3778108`) reduces unnecessary local graph repair edges and repeat pruning; Wolverine++ already offers fast selective deletion repairs. Any proposal must meaningfully distinguish its memory model, graph semantics, proof and performance.
4. **Code boundary:** `InterInsert_with_expand`, `sync_prune` and `MultiProcess_crng` are present as compiled C++ object code in `lib/libconda_internals.a` without ordinary source definitions in the public tree. This prevents line-by-line independent analysis of the exact candidate pathways the most interesting hypotheses depend on.
5. **No baseline reproduction:** environment DNS blocks local clone, so CONDA cannot be built/profiled in this session; locally created fixtures verify only experiment input determinism. No measurements of redundant work, speed, recall or connectivity exist.

## 3. What survived, narrowly

The paper's concluding §7 explicitly identifies a separate possible topic: **search-local deletion repair may fail to discover some affected in-neighbors, leaving stale edges during long runs**. This can be posed as a **replication and stress-testing** question (how does stale-edge fraction vary with deletions and graph density?), not a fresh discovery, because the authors wrote it down. A useful eventual contribution would require a *new mechanism* that provably or empirically recovers missing affected nodes with bounded locality and improved recall at acceptable runtime, and it would need source availability / unmodified baseline reproduction first. It is not a current pass.

## 4. When to reassess

Only reopen after these gates, **in this order**:

1. Obtain a complete permitted, pinned source/executable copy; verify static archive symbol interface and build unchanged CONDA locally.
2. Run matched-set, 1-thread random-order vs cluster-ordered inputs at several seeds; reproduce stock recall@10 and inspect connectivity, no graph invariants broken.
3. Measure how much real time is consumed by repeat per-vertex pruning, and measure actual duplicate distance calls. If the fractional CPU cost is low, **drop**.
4. Explicitly compare any candidate to relevant Greator-style edge selection, Wolverine++ for deletion, and untouched CONDA at equal recall/connectivity. If contribution is already present in competing work, **drop**.
5. Develop a distinct hypothesis with a falsifier, implementation plan in an *isolated scratch copy*, and results across at least five controlled trials. Re-score against the exact rubric. **No code optimization until the profiling gate passes.**

## 5. Professor outreach

**Recommendation: do not send a technical novelty/supervision email today.** The team already knows the clustered-update phenomenon; claiming discovery of it or blaming them for FV's full scan would damage credibility. With no independent runtime evidence, two students would only be proposing exploratory profiling work. That is not prohibited, but it does **not** meet the project's 74-point technical threshold and should not be presented as a research contribution.

If later contact is desired, the scientifically appropriate first communication is a **short, humble request about reproduction or source availability**, clearly identifying which routines are in a compiled archive, without asserting an inefficiency or asking for remote supervision as if a new idea is validated. No message has been drafted for sending, no researcher was contacted, and no GitHub change was made.

**FINAL DECISION: NO-GO**