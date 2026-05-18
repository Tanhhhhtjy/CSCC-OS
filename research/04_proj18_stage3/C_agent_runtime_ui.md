# proj18 Stage 3 — Agent Runtime, Citation Grounding & UI Stack

**Scope.** Validate or refute Stage 2's runtime choices (Anthropic SDK direct + `bge-m3` + Next.js + `react-flow`) for proj18's three-agent pipeline (Extract / Describe / Compare). Decisions below are deliberately opinionated; where Stage 2 was wrong the alternative is named.

---

## 1. Agent runtime: pick the **Anthropic Client SDK (Python) + a thin home-grown orchestrator**, *not* the Claude Agent SDK, *not* LangChain

The shortlist:

| Runtime | Verdict | Why |
|---|---|---|
| **Anthropic Client SDK (Python)** | **Pick this.** | Two LLM call sites (Describe, Compare). No autonomous tool loop is needed — Extract-Agent is pure code. Reproducibility is trivial: we pin model snapshot + `temperature=0` + record raw request/response JSON per repo. |
| Claude Agent SDK (TS/Py) | Reject for the *core pipeline*. | The Agent SDK shines when Claude needs to drive a multi-step tool loop (Read/Edit/Bash/Glob) autonomously. Our Describe-Agent is a *single grounded write* over a pre-built bundle of fingerprint + curated excerpts; an autonomous loop here would *increase* hallucination surface (Claude could wander off and read unrelated files). Acceptable for a *side use*: the Sprint 1 corpus crawler / manifest builder can be a Claude Agent SDK script — there a real tool loop helps. |
| LangChain | Reject. | Heavyweight abstraction tax for two prompts. Versioning churn is the dominant maintenance cost in 4-month student projects; LangChain has shipped 3+ breaking minor releases per year. The "chains of chains" idiom obscures exactly the cache-control + citation-verifier wiring that *is* the load-bearing engineering here. |
| LlamaIndex | Reject. | Optimized for RAG-over-docs ergonomics. We are not doing chunk-and-retrieve over source — that idiom is the exact failure mode the brief warns against. Our retrieval is structural (BM25 on names + tiny embedding index on docstrings). |
| Custom MCP-server-shaped agent | Defer. | Wrapping the Extract-Agent's seven extractors as MCP tools is a *nice-to-have* for demo polish (judges may want to drive the pipeline from Claude Code interactively). Build it in Sprint 7 as a thin shim around the already-stable CLI extractors. Don't make MCP the primary runtime. |

**Architecture in one diagram (text form).**

```
corpus/<year>/<team>/  ──► extract.py (tree-sitter + rust-analyzer)
                            │                                ▼
                            └─► fingerprint.json + excerpts.jsonl
                                                          │
                                ┌─────────────────────────┘
                                ▼
                    describe.py (Anthropic SDK)
                       ├── system prompt (cached, ~6k tok)
                       ├── corpus context (cached, ~4k tok)
                       ├── per-repo fingerprint + excerpts (~3k tok, NOT cached)
                       └── citation verifier (post-hoc, blocking)
                                │
                                ▼ describe/<team>.md  (signed: hash of inputs in frontmatter)
                                │
                                ▼
                    compare.py (Anthropic SDK)
                       ├── MinHash similarity (deterministic, computed first)
                       └── prose only for top-K / bottom-K pairs
```

The orchestrator is ~200 LOC Python. It is a `Makefile` of LLM calls, not a framework. This is the right level of abstraction for a 4-month team.

## 2. Prompt caching strategy

**Cache layout (Claude API, `cache_control: ephemeral`).** Up to 4 breakpoints; we use 3:

```python
system = [
    {"type": "text",
     "text": SYSTEM_PROMPT,                       # ~6k tok, frozen for whole run
     "cache_control": {"type": "ephemeral", "ttl": "1h"}},
    {"type": "text",
     "text": CORPUS_STATISTICS + FINGERPRINT_SCHEMA, # ~4k tok, frozen
     "cache_control": {"type": "ephemeral", "ttl": "1h"}},
    {"type": "text",
     "text": GENRE_GUIDE_FOR_BASE_KERNEL[base],   # ~2k tok per genre (rCore/uCore/scratch)
     "cache_control": {"type": "ephemeral", "ttl": "5m"}},
]
messages = [{"role": "user", "content": render_repo_bundle(repo)}]  # ~3k tok, NOT cached
```

**Why three tiers.** The 1h-TTL prefixes cover an entire 60-repo batch run (≈30 min wall-clock); the 5m genre-tier prefix is shared by all repos of the same base kernel (we sort repos by base kernel so genre cache survives). The per-repo bundle is the only thing that varies.

**Hit-rate math (60 repos, sorted by genre, batch sequential).**

```
Static input per call = 6k + 4k + 2k = 12k tokens
Variable input        = 3k tokens
First call : 12k written + 3k input
Calls 2..60: 12k read   + 3k input
Hit rate   = (59 * 12k) / (60 * 15k) = 78.7%
```

That's ≥80% once we count Compare-Agent calls too (their system prompt is independent but reused 1800 times — those calls hit 99%+).

**TTL math vs 5-min batch.** A 60-repo run at ~30 s/repo describe = 30 min, longer than the 5-min default TTL. We **must** use the 1h TTL for the two big prefixes (cost = 2× input write once, 0.1× input read 59 times — still a huge win vs 60 cold writes). Compare-Agent runs immediately after Describe-Agent; if it shares any prefix, also 1h.

**Pricing verified (Sonnet 4.5, `claude.com/pricing`, May 2026).**

```
Tier                Rate ($/MTok)
─────────────────────────────────
input               $3.00
output             $15.00
cache write 5m      $3.75   (1.25×)
cache write 1h      $6.00   (2.00×)
cache read          $0.30   (0.1×)
```

**Risk: workspace isolation.** Anthropic moved to workspace-scoped cache isolation Feb 5 2026. We must run the whole batch from a single workspace/API key, otherwise we silently miss cache.

## 3. Citation grounding mechanism

**Survey.**

- **Post-hoc verifier** (regex out `file:line` triples, open the file, confirm). Simple, no model coupling, catches *fabricated paths and lines*. Doesn't catch *misattributed quotes* (the path exists but the line says something else).
- **Constrained / grounded decoding** (force generated tokens to come from a vocabulary built from the source). Strong guarantee, but not available in Claude's API; would force a self-hosted model and torch the rest of the design.
- **Guard-tool pattern** (give the LLM a `quote_source(file, start_line, end_line)` tool, ban free-form code in output, require every claim to be paired with a tool call). Works only inside an agent loop (Claude Agent SDK). High accuracy. Slow and expensive.

**Recommendation: hybrid — *prepared excerpts* + *post-hoc verifier*.** Don't let the model invent citations; give it a pre-built `excerpts.jsonl` where each entry has `{id, file, start_line, end_line, text}`, and instruct it to cite by `id`. Then the verifier replays `id → (file, lines)` lookups and confirms the file contents match. Implementation steps:

1. **Extract-Agent emits `excerpts.jsonl`.** Every region the Describe-Agent might cite (trap-frame struct, scheduler function, etc.) is pre-extracted with byte-exact `text`.
2. **Describe-Agent prompt** says: *"You may cite only by `[[ex:42]]` syntax referring to the provided excerpt IDs. Bare file paths in prose are forbidden and will fail validation."*
3. **Verifier** parses the rendered Markdown, resolves every `[[ex:N]]`, opens the repo at `file:start_line-end_line`, computes `sha256` of the file slice, compares to the slice hash stored in `excerpts.jsonl`. Mismatch → fail.
4. **Per-claim text grounding** (optional, Sprint 5 stretch): for each paragraph that cites `[[ex:N]]`, run a small NLI-style check (use `bge-reranker-v2-m3`, free) that the paragraph's claim is supported by the excerpt text. Score < 0.6 → flag for human review.

This keeps the LLM in "writer, not researcher" mode, which is the Stage 2 design discipline.

## 4. Hallucination kill-switch (CI)

A single script run in CI; non-zero exit fails the build:

```python
# tools/verify_citations.py
import json, re, sys, hashlib, pathlib

CITE = re.compile(r"\[\[ex:(\d+)\]\]")

def verify(report_md: pathlib.Path, excerpts_jsonl: pathlib.Path, repo_root: pathlib.Path):
    excerpts = {int(j["id"]): j for j in map(json.loads, excerpts_jsonl.read_text().splitlines())}
    md = report_md.read_text()
    ids = [int(m) for m in CITE.findall(md)]
    if not ids:
        sys.exit(f"FAIL {report_md}: zero citations in describe report")
    errs = []
    for eid in ids:
        ex = excerpts.get(eid)
        if ex is None:
            errs.append(f"unknown excerpt id {eid}"); continue
        f = repo_root / ex["file"]
        if not f.exists():
            errs.append(f"ex:{eid} file missing: {ex['file']}"); continue
        lines = f.read_text(errors="replace").splitlines()
        slice_ = "\n".join(lines[ex["start_line"]-1 : ex["end_line"]])
        h = hashlib.sha256(slice_.encode("utf-8", "replace")).hexdigest()
        if h != ex["sha256"]:
            errs.append(f"ex:{eid} content drift {ex['file']}:{ex['start_line']}-{ex['end_line']}")
    if errs:
        print("\n".join(errs)); sys.exit(1)
    print(f"OK {report_md}: {len(ids)} citations verified")

if __name__ == "__main__":
    verify(pathlib.Path(sys.argv[1]), pathlib.Path(sys.argv[2]), pathlib.Path(sys.argv[3]))
```

CI invocation: a GitHub Action matrix over `corpus/*/*`, each job runs `python tools/verify_citations.py describe/{team}.md excerpts/{team}.jsonl corpus/{year}/{team}`. Any fabricated `[[ex:99]]` or any drift between excerpt and live source aborts the merge. This is the unforgiving gate the Stage 2 risk table demands.

## 5. Vector store: **SQLite + sqlite-vec** (with a fallback note)

For 60 repos × ~100 functions ≈ 6 000 vectors, embedded at 1024-dim (`bge-m3`) = ~24 MB. That fits in **any** of the candidates, so the choice is about ops cost, not capability.

| Option | Verdict |
|---|---|
| **SQLite + sqlite-vec** | **Pick.** Single file, embeds in the repo, zero infra. Status as of May 2026: still pre-v1 (v0.1.10-alpha). No HNSW; brute-force KNN. At 6k vectors brute force is sub-millisecond, irrelevant. The pre-v1 warning is real but our schema is trivial (id, vec, metadata) and any breaking change is a one-day port. |
| DuckDB + VSS | Reasonable; VSS is HNSW-backed and DuckDB is well-loved for analytics. But we'd carry a heavier dep for no perf win at this scale. |
| Qdrant | Overkill. Adds a service to dockerize. |
| pgvector | Overkill *and* slowest to set up. |

**Caveat: only embed function-level docstrings and identifiers**, never raw code bodies. Code-body similarity correlates with syntactic style, not semantics, and pollutes cross-language Rust↔C lookups — Stage 2 already committed to this discipline; don't soften it.

## 6. Frontend: **Next.js 15 App Router + Tailwind + `visx` for the heatmap + `react-flow` only for call-graph fragments**

Stage 2 said "Next.js + Tailwind + react-flow". The error: `react-flow` is a **node-edge diagram** library (MIT, mature for call graphs). It is *not* a heatmap library — implementing a 60×60 similarity heatmap as react-flow custom nodes is a category mismatch. Split the toolbox:

- **Heatmap (60×60) + 7-fingerprint small-multiples → `@visx/heatmap` and `@visx/xychart`.** visx is Airbnb's "low-level d3 + react" set; perfect for publication-quality, deterministic SVG charts that we want to also export to the Sprint 8 paper.
- **Call-graph fragment in the per-repo detail view → `react-flow`.** This is what it's built for.
- **Per-pair diff view → `react-diff-view` or `monaco-diff`.** Monaco is heavier but lets judges click into the source.

**Framework: Next.js 15 App Router, stable since Oct 2024.** React 19 + Server Components let us render the heavy heatmap (precomputed JSON) entirely on the server and stream it; the client bundle stays small. Caveat: Next 15 changed caching defaults to *uncached*; for a static analysis dashboard we *want* the old behavior on data routes — set `export const dynamic = 'force-static'` on the report routes. Tauri/Vite alternatives are rejected: judges need a hostable URL, not a desktop binary.

**State management.** Don't introduce Redux/Zustand for a read-only dashboard. URL params + React Server Components hold all the state that matters (`/repo/[team]`, `/compare/[teamA]/[teamB]`). One `useState` per filter widget.

**Hard discipline (per Stage 2 risk #6 "Dashboard becomes the project").** Sprint 7 ends with a working *static export*: `next build && next export` produces `out/` that judges can `cd out && python -m http.server`. Live submission UI is *optional* on top of this.

## 7. Reproducibility / replay

A judge must be able to re-run our analysis offline with one command. **Recommendation: Docker Compose for the pipeline, with a manifest-pinned corpus.** Nix would be more rigorous but the team's Nix fluency is zero (Sprint 1 risk).

```
proj18/
├── docker-compose.yml          # extractor + dashboard services
├── Dockerfile.extractor        # python 3.12 + tree-sitter + rust-analyzer pin
├── Dockerfile.dashboard        # node 22 + next 15
├── corpus.lock.json            # {team: {git_url, commit_sha}} for all 60 repos
├── models.lock.json            # {bge-m3: hf_revision_sha, claude: "sonnet-4-5-20251022"}
└── make replay                 # clones at pinned SHAs → runs extract → runs describe → renders
```

The `replay` target uses cached LLM outputs by default (we ship `describe/*.md` + `compare/*.json`) so a judge sees the exact figures from our submission without spending API credits. A `make replay-from-scratch` flag (requires `ANTHROPIC_API_KEY`) re-runs the LLM stages. This is the academic-credibility move: the structural pipeline is fully reproducible offline; the LLM prose is reproducible *given* an API key and `temperature=0`.

## 8. Cost ceiling

**Workload.**
- Describe: 60 repos × ~15k input + ~4k output per call.
- Compare: ~1800 pair-prose calls (we only narrate top-K + bottom-K per dimension; pessimistic 30/dim × 7 dim ≈ 200 narrated pairs; let's bound at 1800 to be safe). Each ~8k input + ~1k output.

**Pessimistic (no caching, all on Sonnet 4.5).**

```
Describe: 60 × (15k × $3 + 4k × $15) / 1e6 = 60 × (0.045 + 0.060) = $6.30
Compare : 1800 × (8k × $3 + 1k × $15) / 1e6 = 1800 × (0.024 + 0.015) = $70.20
Total                                                                ≈ $76.50
```

**Optimistic (78% cache hit on Describe, Compare on Haiku 4.5, narrate only top/bottom-K).**

```
Describe (cache hit):
  first call : 12k × $6.00 + 3k × $3 + 4k × $15 = $0.072 + $0.009 + $0.060 = $0.141
  next 59    : 12k × $0.30 + 3k × $3 + 4k × $15 = $0.0036 + $0.009 + $0.060 = $0.0726 each
  total ≈ $0.14 + 59 × $0.073 = $4.45
Compare on Haiku 4.5 (input $1, output $5), 200 pairs narrated:
  200 × (8k × $1 + 1k × $5)/1e6 = 200 × $0.013 = $2.60
Total                                                                ≈ $7.05
```

**So the band is roughly $7–$77 for one full corpus run.** Across 8 sprints with iteration we expect 5–10 full runs ⇒ project LLM bill ~$35–$770. Stage 2's "$50 vs $500" estimate was about right.

**Tier per task.**

| Task | Model | Why |
|---|---|---|
| Describe-Agent | **Sonnet 4.5** | Reports are the deliverable; quality matters; budget allows. |
| Compare-Agent prose | **Haiku 4.5** | Output is short, formulaic ("X uses Sv39 while Y uses Sv48"). 3× cheaper input, 3× cheaper output. Quality drop is invisible at this output length. |
| Corpus-crawler agent (Sprint 1, Claude Agent SDK) | **Haiku 4.5** | Tool loop driving git clones. |
| Citation verifier NLI step (optional) | local `bge-reranker-v2-m3` | $0. |

**Push-back on Stage 2:** the original plan didn't tier — assumed Sonnet across the board. Mixing in Haiku 4.5 for the Compare prose cuts cost ≈10× on that stage and lets us spend the headroom on more iterations.

---

## Summary of changes vs Stage 2

| Stage 2 | Stage 3 verdict |
|---|---|
| Anthropic SDK direct + custom orchestration | **Confirmed.** Reject Agent SDK / LangChain for the core loop; allow Agent SDK for the Sprint 1 crawler. |
| Prompt caching ≥80% hit rate | **Confirmed.** Concrete 3-breakpoint layout, 1h TTL for the two big prefixes, sort batch by base-kernel. |
| Citation grounding | **Refined.** Pre-extracted `excerpts.jsonl` cited by ID + post-hoc hash verifier in CI. No free-form file paths in prose. |
| `bge-m3` for retrieval | **Confirmed.** Embed docstrings + identifiers only, not bodies. |
| Vector index unspecified | **Added.** sqlite-vec; brute force is fine at 6k vectors. |
| Next.js + Tailwind + react-flow dashboard | **Split.** `visx` for heatmap & small multiples (where the marks-of-rigor live); `react-flow` only for call-graphs; Monaco diff for pair drilldown. |
| Reproducibility unspecified | **Added.** Docker Compose + corpus.lock + cached LLM outputs shipped in repo; `make replay` works offline. |
| Single-model cost | **Refined.** Sonnet for Describe, Haiku for Compare prose. Optimistic per-run $7, pessimistic $77. |
