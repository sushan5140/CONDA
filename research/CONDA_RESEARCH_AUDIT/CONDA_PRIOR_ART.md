# CONDA — Prior-art and novelty screen (through 2026-10-18)

**Scope:** dynamic approximate nearest-neighbor proximity graphs; candidate reuse, batch updates, reverse-edge repair, connectivity-aware pruning, and concurrency. Sources below are actual linked publications. **Peer-reviewed status is explicitly distinguished from preprints.** This is a targeted relevance audit, not a claim to have systematically reviewed every paper published worldwide.

## Publication metadata checked

**Target:** Darae Lee and Min-Soo Kim. **CONDA: A Connectivity-Aware Dynamic Index for Approximate Nearest Neighbor Search over Evolving Data.** *Proceedings of the VLDB Endowment* **19(11), 3357–3370 (2026)**. Publisher PDF https://www.vldb.org/pvldb/vol19/p3357-lee.pdf ; DOI https://doi.org/10.14778/3836663.3836694 . Paper §4.2–4.4 covers CRNG pruning, in-neighbor expansion, lazy deletion; §5.4.2 openly acknowledges clustered-update re-pruning; §7 explicitly identifies missed affected in-neighbors/stale edges as an open direction.

## High-priority competing work

| Work / verifiable URL | Status by October 2026 | Contribution relevant to CONDA | Consequence for proposed hypotheses |
|---|---|---|---|
| **FreshDiskANN / FreshVamana**, Aditi Singh et al. https://arxiv.org/abs/2105.09613 | 2021 arXiv preprint; baseline family implemented in Microsoft DiskANN. Avoid claiming a peer-reviewed publication based on this URL alone. | Streaming graph insertion, deferred batched consolidation and graph repair, with trade-off between update costs and quality. | Batched maintenance and consolidation are **old**; cannot claim batching alone is novel. |
| **IP-DiskANN / IPVamana**, Haike Xu et al., *In-Place Updates of a Graph Index for Streaming Approximate Nearest Neighbor Search*. https://arxiv.org/abs/2502.13826 | 2025 arXiv; a VLDB 2026 workshop program also lists the work; verified sources do **not** establish a main-conference peer-reviewed proceedings paper. | In-place insertion and deletion without large consolidation batches, search-local replacement of missing incoming edges, addressing singly linked graph deletion. | Deletion neighborhood repair and scan avoidance are established. Not direct evidence that CONDA first-stage deletion overhead is small. |
| **Wolverine, Wolverine+, Wolverine++**, Dawei Liu, Bolong Zheng, Ziyang Yue, Fuhao Ruan, Xiaofang Zhou, Christian S. Jensen, *PVLDB* **18(7):2268–2280 (2025)**. https://doi.org/10.14778/3734839.3734860 | **Peer reviewed.** | Repairs monotonic paths during deletions; + limits candidate space to two hops; ++ selects higher-quality repair candidates. | Strong prior art for selective local deletion repair. Updating fewer edges is not itself a novel contribution. Paper §5.4 shows Wolverine++ is substantially faster on clustered updates, but with lower search quality. |
| **Greator** (Song Yu et al.), *A Topology-Aware Localized Update Strategy for Graph-Based ANN Index*, *PVLDB* **19(3):495–508**, journal issue bearing 2025 date; presented at VLDB 2026. https://www.vldb.org/pvldb/vol19/p495-yu.pdf and DOI https://doi.org/10.14778/3778092.3778108 | **Peer reviewed.** | Localizes update impact and disk-page writes; similarity-aware local connection specifically avoids inserting redundant candidate edges and triggering costly repeat neighbor pruning. | **Direct overlap with H1**. Distinct disk-resident implementation from in-memory CONDA, so it does not automatically invalidate every CONDA-specific variant; nevertheless, a generic “batch or selective pruning of redundant edges” pitch is not novel. |
| **CleanANN / CleANN**, Ziyu Zhang, Yuanhao Wei, Joshua Engels, Julian Shun, *SPAA 2026*, pp. 247–260. https://doi.org/10.1145/3816782.3819219 ; preprint https://arxiv.org/abs/2507.19802 | **Peer reviewed** under final proceedings title **CleanANN: Efficient and Robust Full Dynamism in Graph-based Approximate Nearest Neighbor Search**. Earlier arXiv/repository uses **CleANN**. Note: not the unrelated CLEANN lock-free *tree* paper from SPAA 2025. | Workload-aware linking, query-adaptive on-the-fly consolidation and semi-lazy cleaning for concurrent fully dynamic ANN. | Important challenge to novelty of partial/lazy repair and concurrency maintenance; distinct topology and query-activity policies. |
| **SPFresh**, Yuming Xu et al., *SOSP 2023*, pp. 545–561. https://doi.org/10.1145/3600006.3613166 | **Peer reviewed.** | Incremental in-place updates for billion-scale vector search, update locality and clustered/segment structure. | Broader evidence that selective local updates are mature; not a direct implementation of CRNG. |
| **Proximity Graph Maintenance for Fast Online Nearest Neighbor Search** (IPGM), Zhaozhuo Xu et al. https://arxiv.org/abs/2206.10839 | **Preprint** (not independently verified as a refereed venue). | General incremental graph maintenance for online insertions and deletions. | Defeats claim that incremental graph repair is novel in itself. |
| **NSG**, Cong Fu et al., *PVLDB* **12(5):461–474 (2019)**. https://doi.org/10.14778/3303753.3303754 | **Peer reviewed.** | Navigating spreading-out graph; graph connectivity and neighborhood formation for ANN; baseline ancestry. | Connectivity preservation existed in static settings, although CONDA's local two-hop rule differs. |
| **DiskANN**, Suhas Jayaram Subramanya et al., *NeurIPS 2019*. https://www.microsoft.com/en-us/research/project/project-akupara-approximate-nearest-neighbor-search-for-large-scale-semantic-search/publications/ | **Peer reviewed**, official Microsoft Research project publication listing linked; conference-paper URL not independently checked. | Robust pruned graphs, bounded degree, scalable ANN baseline. | Distance-based pruning and graph inter-insertion already widely used. |

**Source note:** For DiskANN, use the verified Microsoft Research project bibliography linked in the table and the original CONDA paper's references until a direct publisher link has been checked.

## Functional comparison by idea

### H1 — Batched or deferred graph maintenance

**Already done elsewhere:** FreshDiskANN batches deletion consolidation; Greator reduces redundant repair edges/re-pruning; IP-DiskANN and Wolverine do localized repair. **Already acknowledged by target:** CONDA §5.4.2 says correlated arrivals lead to repeated re-pruning. `BatchInsert_crng` in public source is merely parallel per-point calls, not proof the authors overlooked the workload. A novel CONDA-specific coalescing algorithm would need to show its state consistency, connectivity conditions and safe concurrency boundaries, and outperform Greator-like selective update adapted to memory.

**Decision:** no sufficient originality evidence. Retain only as a **measurement question**.

### H2 — Candidate reuse or cached pairwise distances across similar insertions

Greedy graph search and local candidate selection are longstanding. CONDA §4.2 explicitly avoids pairwise distances outside already-connected candidates, so naive proposed cache may have little reuse. None of the verified competing papers above was proven to provide *the exact same cross-insert CRNG distance memoization*, so we must **not** claim complete prior-art identity. But vague locality/caching, without counts, collision/validity proof, and correct invalidation under edge rewiring, is not a defensible new research contribution.

**Decision:** speculative; novelty remains **unproved**, not definitively ruled out. Cannot pitch without real repeated-operand counts and memory overhead results.

### H3 — Faster deletion scanning

Greator addresses disk scan avoidance; IP-DiskANN and CONDA design local deletion. Crucially, `src/index.cpp:322–387`'s `O(NR)` scan is benchmark **FV**, whereas actual CONDA dispatch is `MultiProcess_crng` (opaque) + `ConsolidateDelete_conda` (visible deleted-set cleanup, no full scan). Therefore claiming a newly identified CONDA global-scan bottleneck is **contradicted by the dispatch**. A careful study could quantify hidden local repair costs, or missed in-neighbors/stale edges as admitted in CONDA §7, but this is distinct from removing a scan.

**Decision:** reject the original H3 mechanism.

## Citations for source/paper verification

- Target author affiliations and bibliography: https://www.vldb.org/pvldb/vol19/p3357-lee.pdf pp. 3357 and 3366–3370.
- Conference listing of CONDA and Greator: https://vldb.org/2026/program.html .
- DBLP target metadata: https://dblp.org/rec/journals/pvldb/LeeK26a .
- DBLP Wolverine venue: https://dblp.org/rec/journals/pvldb/LiuZYRZJ25 .
- DBLP peer-reviewed CleanANN: https://dblp.org/rec/conf/spaa/ZhangWES26 .
- Greator paper / content: https://www.vldb.org/pvldb/vol19/p495-yu.pdf .
- SPFresh in target paper bibliography §References [58]; its DOI listed above should be rechecked prior to scholarly submission.

### Novelty policy

An idea only passes if the algorithmic distinction is specified, its novelty relative to the **same** memory/disk/concurrency setting is defended, controlled performance evidence exists, and correctness/recall/graph connectivity are not weakened. None of H1–H3 satisfies these gates as of this audit.