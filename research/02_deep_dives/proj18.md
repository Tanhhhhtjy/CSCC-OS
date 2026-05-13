# proj18 Feasibility Report — Small-OS Analysis & Comparison Agent

**Track:** 2026 CSCC OS Functional Challenge (A-level, academic)
**Deliverable:** LLM agent that describes & compares CSCC OS-kernel-track submissions (2021-2025) against a new entrant
**Tutor:** Shao Zhiyuan (HUST)
**Stage 1 score:** 23/25

---

## 1. Technical path

The trap with this topic is to slide into "ChatGPT over a git repo": embed the source, RAG over chunks, ask GPT to summarize. That gets us a B+ at best, because every reviewer can do the same thing in an afternoon. The differentiation must come from **OS-specific structural fingerprints** — features no general code-RAG tool extracts.

We propose a **three-stage pipeline**, each stage producing a typed artifact that the next stage consumes. The pipeline is deliberately *not* end-to-end LLM; the LLM is the orchestrator and the writer, but the structural extraction is symbolic.

**Stage A — Ingest & normalize.** Crawl the CSCC OS-kernel-track public corpus: gitlab.eduxiji.net hosts the canonical submissions (`OSKernel2021-LinanOS`, `OSKernel2022-Oops`, `OSKernel2022-X`, `OSKernel2022-Byte OS`, `OSKernel2023-umi`, plus the 2024-2025 cohorts), and the GitHub mirror exists for several of these. `git clone --depth 0` everything, normalize to a `corpus/<year>/<team>/` tree, write a `manifest.json` with detected base kernel (rCore? uCore? from-scratch?), language mix (`tokei`), and entry-point path. Detection rules: presence of `os/src/sbi.rs` + `linker.ld` + rCore-style chapter naming → rCore-derived; `kern/init/init.c` → uCore-derived; otherwise heuristically scored.

**Stage B — Structural fingerprint extraction.** This is the project's intellectual core. We extract **seven OS-specific fingerprints**:

1. **Boot path** — symbolic trace from the linker entry symbol (`_start`) through the first three function-call edges to either `rust_main` or `kern_init`. Encoded as a directed graph fragment + asm-vs-Rust ratio.
2. **Trap frame layout** — find the `TrapContext`/`trapframe`/`struct trap_frame` definition, extract the field order and width (`x[32], sstatus, sepc, ...`), normalize to a canonical 33-tuple, hash with a position-sensitive MinHash.
3. **Scheduler** — find the function called from the timer-interrupt handler; classify into `{round-robin, MLFQ, CFS-like, BFS, custom}` via a hand-written rule set on the AST shape (loops over a runqueue, presence of priority-comparison expressions, etc.).
4. **Memory management** — page-table walk depth (Sv39 vs Sv48), allocator class (`buddy_system_allocator` crate? slab? bump?), copy-on-write present/absent.
5. **File system** — easy-fs derivative? FAT32 port? from-scratch? Detected by superblock layout literals + inode struct shape.
6. **IPC** — pipe + signal + mailbox + futex matrix, syscall-number → handler-function map.
7. **Syscall surface** — full enumeration of `SYSCALL_*` constants and their handler functions, normalized to Linux syscall numbers where applicable.

Extraction toolchain: **tree-sitter for both C and Rust** (both grammars are upstream-mature and we drive them from one Python process via `py-tree-sitter`); a thin **custom AST visitor** to lift struct shapes and call edges into a CPG-lite graph (we explicitly *reject Joern* — it does C/C++/Java/Python/Kotlin per its repo description but **not Rust**, which is half the corpus). For cross-language call-graph stitching we use `rust-analyzer --print-call-graph` on the Rust side and `clang -emit-ast` + a small Python walker on the C side, then union the graphs at the syscall boundary. CodeQL is rejected for the same Rust-coverage reason.

Each repo produces a `fingerprint.json` (~5KB) — *this* is the embedding-ready, human-readable artifact, not source chunks.

**Stage C — Agent orchestration: describe & compare.** Two agents.

- **Describe-Agent**: input = `fingerprint.json` + curated source excerpts (the seven extracted regions, ≤2KB each). Output = a Markdown report with seven sections, each grounded with a file:line citation. We use **prompt caching** (cache the long system prompt + the corpus-level statistical context) — this is where the `claude-api` skill earns its keep; cache hit rate target ≥80%, which roughly halves token cost across the 60+ repos.
- **Compare-Agent**: input = two `fingerprint.json`s + the two describe-reports. Output = a similarity score per fingerprint dimension + a narrative "novelty assessment". The similarity score is *not* LLM-judged for the structural dimensions — it's a deterministic MinHash/Jaccard on the typed fingerprints. The LLM only writes the prose explaining the score. This is the single most important design decision: **deterministic structure for accuracy, LLM for prose**, never the inverse. It eliminates the "hallucinated similarity" failure mode the brief explicitly warns against.

A **retrieval index** sits underneath: BM25 over file paths and function names (fast literal recall), plus a small embedding index (bge-m3 or Qwen3-Embedding, both free/open) over function docstrings only — *not* over raw code, because raw-code embedding similarity is dominated by syntax noise and famously misleads on cross-language comparison.

A **dashboard** (Stage D, soft deliverable) renders the seven fingerprints as small-multiples + a heatmap of cross-year similarity. Built with `frontend-design:frontend-design` skill — Next.js + Tailwind + `react-flow` for the call-graph fragments.

## 2. Four-month milestone plan (8 sprints × 2 weeks)

**Sprint 1 (W1-2) — Corpus acquisition & normalization.** Crawl gitlab.eduxiji.net + GitHub mirrors, build `corpus/` with ≥40 repos spanning 2021-2025, write `manifest.json` schema, hand-label base-kernel for 10 repos as ground truth. Deliverable: `make corpus` reproducible.

**Sprint 2 (W3-4) — Tree-sitter extraction harness.** `py-tree-sitter` driver for C and Rust, function/struct/call-edge extractors, golden-file tests against three hand-annotated repos. Acceptance: parses 100% of Rust files, ≥98% of C files (kernel-style C usually clean, but watch for inline asm and Linux-style `#include` cycles).

**Sprint 3 (W5-6) — Boot-path + trap-frame fingerprints.** First two of the seven extractors, end-to-end on 5 repos. Run cross-validation: are the auto-extracted trap frames identical to hand-annotated ones? Target: ≥95% field-order match. Sprint exit gate: `make fingerprint corpus/2022/Byte_OS` succeeds in <30s.

**Sprint 4 (W7-8) — Scheduler + MM + FS + IPC + Syscall extractors.** The remaining five fingerprints. Higher uncertainty here — schedule and MM classification rules need iteration. Time-box: if any single fingerprint takes >3 days, ship a degraded version (e.g., "unknown, manual review" classification) rather than slip the sprint.

**Sprint 5 (W9-10) — Describe-Agent + prompt caching.** Claude API integration via the `claude-api` skill, cached system prompt + cached corpus statistics. Generate describe reports for all 40+ repos. Run a human spot-check: 10 reports, blind-rated for factual accuracy by a teammate who has read the source. Hard gate: ≥9/10 factually correct, zero hallucinated file paths.

**Sprint 6 (W11-12) — Compare-Agent + MinHash similarity.** Pairwise similarity matrix across the corpus (≈40×40 / 2 = 780 pairs, cheap because the structural part is deterministic). LLM prose only for the top-K and bottom-K pairs per dimension. Validate against intuition: the 2022/2023 rCore-derivatives should cluster.

**Sprint 7 (W13-14) — Dashboard + novelty mode.** Next.js + Tailwind dashboard, "submit a new repo" workflow: drop a tarball, run extraction + compare, get the report. This is the *judges' day* deliverable.

**Sprint 8 (W15-16) — Hardening, paper, video.** Run `paper-orchestra` skill to draft a 6-page workshop-style write-up (this is an A-level academic topic; a paper artifact lifts the academic-credibility score). Record a 7-min demo. Freeze tag.

## 3. Skills inventory & gap analysis

**Have:** Python, JavaScript/TypeScript, tree-sitter familiarity (one teammate), LLM API usage, basic Tailwind. **Gap:** rust-analyzer's call-graph output format (half a day to learn); MinHash + LSH library choice (`datasketch` is mature, decision is trivial); Next.js streaming for the dashboard (one teammate spends Sprint 1-2 evenings on it).

Skills we lean on directly:
- **`claude-api`** — mandatory. Prompt caching is the difference between a $50 and a $500 corpus run; cache the 15KB system prompt and the per-repo style guide.
- **`paper-orchestra`** — for the Sprint 8 write-up. The A-level / academic tag genuinely rewards a publication-style artifact.
- **`frontend-design:frontend-design`** — for the dashboard. Without it the visuals will read as generic and lose the "human-friendly output" grading criterion.
- **`superpowers:test-driven-development`** — for the seven extractors; each has a golden-file form factor.

**New skills worth authoring:**
- `os-structural-fingerprint` — codifies the seven-dimension fingerprint schema, the canonical trap-frame normalization, the scheduler-classification rule set. Reusable across this project, future CSCC years, and any "OS-kernel comparison" research downstream.
- `cscc-corpus-crawler` — a small skill that knows the gitlab.eduxiji.net URL conventions and the GitHub mirror discovery heuristics. Small but saves every new analyst a day.

## 4. Demo + evaluation plan

**Demo:** Live at judging — paste a new repo URL into the dashboard, watch fingerprint extraction stream (≤2 min), get the describe-report rendered with cited line numbers, see the similarity heatmap update with the new column highlighted. Click any cell → see the per-dimension Jaccard score + the LLM's prose explanation + the source-snippet diff.

**Evaluation:**

| Metric | Method | Target |
|---|---|---|
| Describe factual accuracy | blind human rating, n=20 reports, 5 graders | ≥95% statements with valid file:line citation |
| Hallucinated file/function rate | grep cited paths against actual repo | 0% |
| Fingerprint extraction coverage | over 40-repo corpus | ≥90% of repos with all 7 dimensions resolved (not "unknown") |
| Compare-Agent agreement with experts | 30 expert-labeled pairs, Cohen's κ | κ ≥ 0.7 |
| End-to-end wall-clock for a new repo | from URL paste to rendered report | ≤180s |
| Prompt-cache hit rate | per `claude-api` skill telemetry | ≥80% |

The describe accuracy + zero-hallucinated-citation targets directly answer the brief's explicit demand ("尽量避免大模型幻觉；比较文档应做到精准无误"). The κ ≥ 0.7 metric is the academic gloss — quote it in the paper.

## 5. Risks & mitigations

- **"ChatGPT for code" critique.** This is the existential risk for an academic topic. Mitigation: every public artifact must lead with the *structural* fingerprint (a graph, a table) before any LLM prose; LLM is decoration, structure is substance. Reviewers must immediately see the OS-specific abstractions.
- **Corpus access / licensing.** gitlab.eduxiji.net repos are mostly open but inconsistently licensed. Mitigation: only show derived fingerprints + cited snippets in any public artifact, never re-host full source; document the policy in `CORPUS_POLICY.md`.
- **Rust + C dual-language extractor maintenance burden.** Mitigation: keep extractors *thin* and language-agnostic at the schema level (the seven fingerprints have language-agnostic representations); push language specifics into per-language adapters.
- **LLM hallucinated citations.** Mitigation: post-validate every `file:line` in the describe-report against the actual repo, drop or red-flag any unverifiable claim before rendering. This is a strict gate, not a soft one.
- **Joern/CodeQL temptation.** Mitigation: don't. Neither covers Rust adequately as of 2025; building on them creates a non-Rust gap nothing else can fill. Already decided in Section 1, must hold the line.
- **Dashboard becomes the project.** Mitigation: Sprint 7 has a hard 2-week budget; if behind, ship a static-site report-per-repo and skip live submission.

## 6. Score & rationale

**Final score: 7.5/10.**

Why this high: the OS-specific structural fingerprint approach is genuinely defensible against the "ChatGPT over a repo" critique, the A-level academic tag rewards exactly this kind of rigor, the data exists and is accessible, the tool stack (tree-sitter + Claude API + Next.js) is boring and proven, and the team's existing skill set maps cleanly. The Stage 1 23/25 score reflects judge appetite for "agent applied to OS pedagogy".

Why not higher: (a) **Differentiation is the load-bearing assumption** — if a competing team independently lands on the same seven-fingerprint idea (and several will try), our edge collapses to execution quality and dashboard polish, both of which are easier to draw than to ship. (b) The success criterion ("精准无误") is *unforgiving* — one fabricated citation in the judges' randomly chosen report is a B-grade exit. (c) Corpus crawling has political friction we cannot fully predict (some repos may go private, eduxiji.net rate-limits). (d) Compared to proj61, the upside ceiling is similar but the floor is lower: proj61 always produces a runnable kernel; proj18 can produce a beautiful report system that the judges decide is "just RAG".

**Recommendation: green-light, but only if the team commits to the *no-LLM-in-the-structural-loop* discipline. If that discipline slips in Sprint 4-5, kill it by Sprint 6 rather than deliver a glossy hallucination engine.**
