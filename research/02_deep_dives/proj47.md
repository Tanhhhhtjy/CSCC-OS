# proj47 — Vector Retrieval I/O Optimization for Agent Memory

Stage-2 deep dive (storage-expert, team cscc-os-stage2). Stage-1 score 23/25, level B,
academic-type. Mentor: 崔立骁 (Nankai). The brief asks for an SSD-resident, dynamic,
high-concurrency vector retrieval engine that survives a memory budget of 10–20 % of
the dataset while delivering recall@10 ≥ 0.85 and LangChain-compatible APIs.

## 1. Problem Framing and Why This Is an OS Problem

An LLM agent's "long-term memory" is naturally modeled as a vector store of embedding
vectors plus payload. Each turn produces new memories (writes) and recalls similar
past memories (reads). Two pathologies emerge once the corpus outgrows RAM:

1. **Graph traversal kills the page cache.** Both HNSW and Vamana (DiskANN's graph)
   visit O(log N) nodes per query, but the visited nodes are scattered by design —
   small-world graphs only work because edges jump randomly. The kernel's
   readahead heuristic prefetches sequential pages it will never use, while the
   cold pages we actually need page-fault on the critical path. On SIFT1M
   (128-d, 1 GB raw) a naive mmap'd HNSW yields < 200 QPS at 100 MB RSS because
   every hop is a 4 KiB read followed by a 100 µs SSD service time.

2. **Real-time inserts cause write amplification.** Graph indexes are not LSM
   trees: an in-place edge update rewrites a whole node (≈ 4 KiB for SIFT, more
   for higher-dim datasets) and the node has to be re-fsync'd to survive crashes.
   Worse, a write can mutate the in-neighbor list of every neighbor — DiskANN's
   reference build does this with a global lock. SPFresh (SOSP'23, Microsoft)
   showed that ignoring this leads to either stale graphs or > 30× write
   amplification.

The OS layer can fix both with the right primitives: a user-space cache that
understands graph hop locality, an io_uring + NVMe passthrough fast path that
hides 80 µs of SSD latency behind the in-CPU distance computation, and an
LSM-flavored writer that turns random edge mutations into sequential log appends
plus periodic compaction.

## 2. State of the Art (2024–2026)

- **DiskANN (microsoft/DiskANN).** As of v0.52.0 (May 2026) the repo is 99.9 %
  Rust under MIT. The original 2019 NeurIPS paper introduced Vamana: a flat
  single-layer graph with bounded out-degree R and a beam-search frontier of
  width L. Vamana was *designed* for SSD: R is large (typically 64–128) so the
  graph fits in fewer hops; "long-range" pruning edges keep the diameter low
  even at billion scale; PQ-compressed vectors live in RAM as the rerank stage,
  while full vectors and adjacency lists live on SSD and are fetched only when
  visited. The Rust rewrite means we can statically link DiskANN crates and
  swap our own `DiskBackend` trait — perfect for a competition fork.

- **SPFresh (SOSP'23).** Builds on SPANN (cluster-based, not graph-based) and
  introduces the LIRE in-place rebalance protocol: when a posting list grows
  past a threshold, vectors are reassigned to neighbor clusters one at a time
  under a local lock instead of rebuilding the whole index. Reported up to
  10× lower update tail-latency than DiskANN-update at iso-recall. The
  cluster-based design is easier to update than a graph but slightly less
  accurate at low memory budgets.

- **Milvus DiskANN backend.** Productionized DiskANN as a Knowhere index,
  exposes `search_list` and `beamwidth` as tuning knobs, and uses mmap by
  default. Performance is bottlenecked by the kernel page cache exactly as
  this project predicts.

- **Qdrant on-disk HNSW.** Stores the HNSW graph + payload on disk via mmap;
  recommends giving the OS plenty of free RAM — i.e. punts the cache problem
  to the kernel. Their docs explicitly warn that recall degrades at low RAM
  budgets, confirming the gap this project targets.

- **vLLM PagedAttention.** Not vector search, but the technique transfers:
  treat the index as 4 KiB pages with a software-managed page table and an
  LRU/LFU-like reclaimer. The lesson is that *user-space paging beats kernel
  paging when the access pattern is application-specific*, which is exactly
  our case.

- **io_uring + NVMe passthrough.** `IORING_OP_URING_CMD` with the
  `/dev/ng0n1` char device skips the block layer, uses fixed buffers, and on
  Linux 6.1+ delivers ~5 µs syscall-to-completion latency with IRQ-less
  polling. For a graph that issues 32–64 random 4 KiB reads per query, this
  alone is a 2–3× speedup over `pread()`.

## 3. HNSW vs Vamana: We Pick Vamana

| Axis | HNSW | Vamana (DiskANN) |
|---|---|---|
| Layers | Multi-layer hierarchical | Single flat layer |
| Out-degree | Small (M ≈ 16–32), levels add up | Large (R ≈ 64–128) |
| Build cost | Lower (random insertion) | Higher (two-pass with α-pruning) |
| SSD friendliness | Poor: entry-point pages re-read every query | Good: a 4 KiB block holds one node + its edges |
| Diameter | log₂N hops | ~6–8 hops on 1 B vectors |
| Updates | Insert easy, delete needs tombstone + repair | Same, but SPFresh-style protocols exist |
| Memory-mapped baseline | Mature (hnswlib) | Mature (DiskANN) |

We choose **Vamana** because:

1. **One node per 4 KiB block.** With R = 64 and 128-d vectors at fp32, a node
   is 64×4 + 128×4 = 768 B; including a 32 B header we comfortably fit one
   node per 4 KiB sector. Each beam-search hop is exactly one NVMe read —
   no read amplification, no cross-block edge chasing.
2. **Diameter is the figure of merit on SSD.** HNSW's upper layers help in RAM
   but become latency multipliers on SSD (every layer transition is a cache
   miss). Vamana's flat layout means hops = SSD reads, full stop.
3. **DiskANN crates are Rust + MIT** — we can vendor `diskann-disk` and replace
   only the `IoBackend` and `Cache` traits without forking the whole stack.
4. **PQ-in-RAM, full-vector-on-SSD split is built in.** Gives us a natural
   memory budget knob: PQ codebook + compressed vectors ≈ 5–10 % of dataset,
   which leaves headroom inside the 10–20 % budget for a hot-node cache.

The cost is that HNSW has a simpler insert path. We mitigate by adopting
SPFresh-style local-rebalance for inserts and an LSM-style log for deletes,
described in §4.

## 4. Proposed System Design

```
         ┌────────────────── LangChain VectorStore API ──────────────────┐
         │                                                                │
         │  add_texts()    similarity_search_with_score()    delete()     │
         └───────┬──────────────────┬─────────────────────────┬───────────┘
                 ▼                  ▼                         ▼
          ┌─────────────┐  ┌──────────────────┐    ┌─────────────────┐
          │  WAL writer │  │  beam-search     │    │  tombstone log  │
          │  + memtable │  │  scheduler       │    │                 │
          └───────┬─────┘  └────────┬─────────┘    └────────┬────────┘
                  │                 ▼                       │
                  │    ┌──────────────────────────┐         │
                  │    │  HOT-NODE CACHE          │         │
                  │    │  hop-aware admit + LFU   │         │
                  │    └────────┬─────────────────┘         │
                  ▼             ▼                           ▼
         ┌────────────────────────────────────────────────────┐
         │           io_uring submission queue (poll mode)     │
         │   fixed buffers, IORING_OP_URING_CMD on /dev/ng0n1  │
         └─────────────────────┬──────────────────────────────┘
                               ▼
                          NVMe SSD
                 ┌────────────────────────────┐
                 │  L0 segments (recent WAL)  │
                 │  L1 Vamana graph (immutable)│
                 │  PQ codes (mmap, lazy)     │
                 └────────────────────────────┘
```

### 4.1 Read path
- Translate query → distance to entry-point seed via PQ codes (in RAM).
- Beam-search with width B (default 8). For each frontier expansion, submit
  up to B `URING_CMD` reads in parallel; while NVMe services them, compute
  PQ distances on already-visited candidates to **overlap I/O with compute**.
- Cache lookup before each submit; cache miss → admit on second touch only
  (avoids polluting on cold one-shot queries).
- **Hop-aware prefetch.** As soon as a node is fetched, peek at its adjacency
  list and asynchronously fetch the top-k neighbors whose PQ distance to the
  query is below a threshold. This is the "next-hop" prefetch the brief asks
  for; it works because Vamana edges are not random — α-pruned edges point
  toward the query in expectation.

### 4.2 Write path (LSM-flavored)
- New vector → append to WAL (group commit, O_DIRECT + fdatasync every
  16 KiB), insert into in-memory memtable (a small Vamana graph capped at
  ~64 K nodes).
- Background thread flushes memtable into a new L0 segment as a compact
  Vamana sub-graph.
- Compaction merges L0 segments into the L1 base graph by running α-prune
  on the union of neighborhoods, similar to SPFresh's LIRE but at the graph
  level. Compaction reads/writes are sequential — the random write storm
  the kernel would otherwise see is fully hidden.
- Deletes are tombstoned (bitmap over node IDs). Compaction garbage-collects
  tombstoned nodes by rewiring their predecessors.

### 4.3 Cache replacement
- Per-node touch counter + last-access epoch.
- Admission policy: **TinyLFU sketch** filters out one-hit cold nodes.
- Eviction: segmented LRU, with the "protected" segment sized to keep the
  graph entry-point's 2-hop neighborhood always resident (≈ R² ≈ 4 K nodes,
  16 MiB on SIFT1M) — these are visited by literally every query.

### 4.4 LangChain integration
- Implement `langchain_core.vectorstores.VectorStore` in Python via PyO3
  bindings. The Rust core exposes `add`, `search`, `delete`, `persist`,
  `from_texts`. Embedding generation stays in user space (any HF model).

## 5. Evaluation Targets (SIFT1M, must-hit numbers)

SIFT1M: 1 M vectors, 128-d fp32, 1 GiB raw. We commit to:

| Metric                | Floor (pass)  | Target (gold) |
|-----------------------|---------------|---------------|
| Recall@10             | ≥ 0.90        | ≥ 0.95        |
| Single-thread QPS     | ≥ 1 500       | ≥ 5 000       |
| 16-thread QPS (NVMe)  | ≥ 12 000      | ≥ 30 000      |
| p99 search latency    | ≤ 10 ms       | ≤ 3 ms        |
| Insert throughput     | ≥ 2 000 ins/s | ≥ 10 000 ins/s |
| Memory budget         | 200 MB (20 %) | 100 MB (10 %) |
| Recall drop after 50 % churn | ≤ 5 pt | ≤ 2 pt        |

Baselines: hnswlib (mmap), DiskANN Rust v0.52.0 (stock), Milvus DiskANN,
Qdrant on-disk HNSW. All measured on a 4-core Xeon-class VM with a single
NVMe Gen-4 SSD (≥ 600 K random read IOPS). For the OS-functional-track judges
we also report **bytes read per query** and **page-cache hit rate**, where
our system should reach > 95 % cache hit by query 1 K on a Zipfian workload.

Mixed-read-write benchmark: YCSB-V (95 % read / 5 % write) and SPFresh's
streaming trace (continuous inserts, 1 M ops). Report read-tail latency
during writer activity — the killer metric for agent memory.

## 6. Risks and Open Questions

1. **DiskANN-rs API churn.** v0.52.0 is recent; trait surfaces may move. We
   pin to a specific tag and vendor the crate. Fallback: hnswlib-rs port +
   DIY Vamana builder (~1500 LOC).
2. **NVMe passthrough requires root + a raw `ng` char device.** Containerized
   judging environments may not expose it. Fallback path: io_uring on the
   regular block device with `IORING_SETUP_IOPOLL`, ~30 % slower but works in
   a VM.
3. **Hop-aware prefetch can over-fetch on adversarial queries** (queries far
   from data manifold). Cap concurrent inflight prefetches per query and
   stop prefetching once beam depth exceeds 2× expected.
4. **LangChain Python overhead** can swamp a 1 ms search. Mitigate with
   PyO3 zero-copy buffers and a batch search API.
5. **Crash consistency.** WAL + tombstone log + segment manifest must
   survive `kill -9`. Plan: per-segment checksum + manifest-as-rename.
6. **PQ training distribution shift** when agent memory drifts. Trigger
   re-training during compaction if measured recall on a held-out probe set
   drops > 2 pt.

A successful submission demonstrates: (a) a working Rust crate plus Python
bindings, (b) FIO-style microbench showing the 4 KiB random read latency
the engine sees end-to-end, (c) recall/QPS tables vs the four baselines on
SIFT1M and ideally GIST1M, and (d) a 30-min agent memory replay (e.g. a
LangChain chat log) showing stable p99 under sustained insert load.
