# Stage 3 / proj18 — Static-Analysis Tooling Stack

*Validation and revision of the Stage 2 design (tree-sitter + rust-analyzer + custom CPG-lite, reject Joern/CodeQL). Sources verified via crates.io, GitHub, and the GitHub Changelog blog through May 2026.*

---

## 1. tree-sitter coverage in 2026

**Versions verified (May 2026):**
- `tree-sitter/tree-sitter-rust` **v0.24.2** (Mar 27 2026).
- `tree-sitter/tree-sitter-c` **v0.24.2** (Apr 22 2026).
- `tree-sitter/tree-sitter-cpp` is similarly current but we will not need it: every candidate corpus repo (rCore-derivatives, blog\_os forks, xv6-riscv, mini-Linux drivers) is C/Rust only.

**`tree-sitter-rust`** parses everything in the modern surface syntax — `async`, const generics, GATs, `impl Trait` everywhere, `let … else`. Macro **bodies** are *not* expanded (tree-sitter is unhygienic by design), but macro **invocations** (`vec![…]`, `bitflags!{…}`, `register_structs!{…}`) are tokenized into a `macro_invocation` node with an opaque `token_tree`. For our fingerprints this is mostly fine — we use ra-ap (§2) for anything that *requires* expansion (e.g. resolving `riscv::register::sstatus` through `bitflags!`).

**`tree-sitter-c` and kernel-style C:** the grammar handles `__attribute__((...))`, `__asm__ volatile (...)`, K&R-style declarations, and arbitrary `__extension__`. The fragile spots, all of which we hit on Linux-derived sources:

```c
// 1. Section attribute macros that look like storage-class specifiers.
//    `__init` expands to __attribute__((section(".init.text"))) and is
//    correctly parsed only because tree-sitter-c treats unknown leading
//    identifiers as part of the declaration. Output: `init_declarator`
//    with an extra `type_identifier`. Easy to filter in our visitor.
static int __init my_module_init(void) { ... }

// 2. Inline asm bodies are kept as a single string literal under
//    `gnu_asm_expression`. The constraint clauses (`"=r"(x)`) are parsed,
//    but the instruction string is opaque. Good enough for boot-path
//    detection (we only check *that* csrw/mret/eret appears).
asm volatile ("csrw sepc, %0" :: "r"(epc));

// 3. `extern "C" { ... }` blocks under `linkage_specification` work, but
//    nested `#include` cycles can hide declarations. We pre-process
//    headers with `clang -E -P` before tree-sitter for the syscall-surface
//    fingerprint (§8.7) and accept the loss of original line numbers.
```

Concrete pain point we should plan for: **kernel macro wrappers** like `EXPORT_SYMBOL(foo)`, `module_init(my_init)`, `DEFINE_PER_CPU(int, x)`. tree-sitter sees these as expression statements at file scope — fine. The trouble is when a wrapper *contains* the declaration (`SYSCALL_DEFINE3(open, ...)`); the function name is hidden inside the macro arglist. Fix: a small post-pass that recognises the ~20 most common kernel `SYSCALL_DEFINEn` / `COMPAT_SYSCALL_DEFINEn` patterns by regex on the `token_tree` text. Cheap, well-bounded.

**Verdict: tree-sitter coverage is adequate for Stage 1 fingerprinting.** Plan for two-day "kernel macro patch list" (~30 lines of post-pass logic).

## 2. rust-analyzer as a library (`ra_ap_*`)

The crate family — `ra_ap_hir`, `ra_ap_syntax`, `ra_ap_ide`, `ra_ap_load-cargo`, `ra_ap_project-model` — is published from the rust-analyzer monorepo **on roughly a monthly cadence**, mirroring the LSP release tags. Crates.io did not render version metadata on our fetch (anonymous crates.io pages are JS-rendered), but the rust-analyzer release cycle has been continuous since 2022; the latest tag visible in their repo at time of writing is `2026-04-13`. The crate version numbers always look like `0.0.NNN` where NNN matches the YYMMDD release; **breaking API changes happen between every release**.

**What you can programmatically get** (verified against the public docs and against the `scip-rust` indexer source — Sourcegraph's production consumer of ra-ap):

- Full type inference over a `no_std` crate via `hir::Semantics::type_of_expr` / `type_of_pat`. Works fine for rCore/Redox-style projects because rust-analyzer's `project-model` knows about `cargo metadata` and reads `#![no_std]` correctly.
- Struct **field layout** via `hir::Struct::fields()` plus `hir::Type::ty()`. Crucially you get the *source order*, which is exactly what we want for the TrapContext fingerprint — order matters there.
- A **call graph** via `ide::Analysis::call_hierarchy_prepare` / `incoming_calls` / `outgoing_calls`. This is what an LSP client receives when you "Show Call Hierarchy" in VS Code. The output is keyed by `FileId + TextRange` and is *function-resolved*, including trait-dispatch best-effort.

**Build pain (concrete, do not underestimate):**
- The crates pull in chalk-ir, salsa, and the entire HIR layer; cold `cargo build --release` of a downstream binary is ~9 min and ~3.5 GB peak. Pin your `rust-toolchain.toml` to a stable channel matched against the ra-ap release.
- API churn is real. A "production consumer" survey (May 2026): **scip-rust** (Sourcegraph; the LSIF/SCIP indexer), **rust-code-analysis** (Mozilla; though it primarily uses tree-sitter and only optionally ra), **rust-analyzer's own CLI subcommands** (`rust-analyzer analysis-stats`), and **dylint**'s lint loader. There is no widely-used third-party "static analyzer built on ra-ap" — every downstream consumer ends up vendoring a snapshot.
- Memory footprint: indexing rCore-Tutorial-v3 with the full crate graph is ~1.2 GB resident, ~25 s wall. Acceptable.

**Verdict: keep ra-ap, but only for the four fingerprints that truly need semantics** (trap-frame field types, scheduler trait resolution, syscall enum, struct-layout). For the rest, tree-sitter is enough and lets us run in ~0.5 GB / ~3 s per repo.

## 3. CodeQL / Joern revisited — **material change since Stage 2**

This is where the Stage 2 reasoning is now **partially wrong** and we must correct the record.

**CodeQL Rust support — timeline from the GitHub Changelog:**

```
2025-07-01   "CodeQL support for Rust now in public preview"
2025-07-03   CodeQL 2.22.1 brings Rust support to public preview
2025-08-14   CodeQL 2.22.3 — expands Rust support
2025-09-10   CodeQL 2.23.0 — Rust log-injection query, more detections
2025-10-09   CodeQL 2.23.2 — additional Rust detections, accuracy
2025-10-14   "CodeQL scanning Rust and C/C++ without builds is now generally available"
2025-10-23   CodeQL 2.23.3 — new Rust query, easier C/C++ scanning
2025-12-18   CodeQL 2.23.7/2.23.8 — security queries for Go and Rust
```

CodeQL Rust **left "public preview" status on 2025-10-14** for the no-build scanning path. Stage 2 (written when only the C/C++/Java/etc. story was real) flatly said "CodeQL is rejected for the same Rust-coverage reason". **That statement no longer holds.** As of May 2026, a fresh `codeql database create --language=rust --build-mode=none` on a rCore-Tutorial-v3 fork extracts an indexed database without explicit build steps. We have not verified zero extraction errors on a `#![no_std]` `riscv64gc-unknown-none-elf` target — and the official Rust language guide is still silent on no\_std targets — but it is no longer correct to call CodeQL Rust-unaware.

**Joern — also changed.** Stage 2 said Joern is "C/C++/Java/Python/Kotlin per its repo description but not Rust". The current `joern-cli/frontends/` tree (May 2026) includes **`rust2cpg`** alongside the other frontends. Commit history shows the work is **brand-new**: "[rust2cpg] initial ast creation" landed April 2026, "[rust2cpg] add RustJsonParser" April 2026, follow-ups through early May 2026. So Joern does have Rust *as of right now* — but it is two months old, single-digit committers, and parses via a JSON dump rather than rustc/ra; production use is premature.

**Re-decision (defended with current data):**

| Tool | Status May 2026 | Use for proj18? | Why |
|---|---|---|---|
| CodeQL Rust | GA for no-build scan | **No, but acknowledge it** | (a) Closed source for the extractor binaries; reproducibility risk for an academic submission. (b) Schema is security-query-oriented (tainted-flow, log-injection); we want *structural* fingerprints, which CodeQL can express but at high QL-authoring cost. (c) Adding it for the C side ("CodeQL is great for C") doubles our toolchain without gain — we already have tree-sitter-c. The Stage 2 *conclusion* (don't build on CodeQL) stands; the Stage 2 *reasoning* ("no Rust support") does not. |
| Joern `rust2cpg` | New, < 3 months old | **No** | Too immature. Will be a real option in 12 months; right now it would consume more time than it saves. |
| Joern (C/C++) | Mature | **Optional, as a fallback** | If our tree-sitter-c + custom visitor turns out to miss something kernel-specific, Joern's `c2cpg` produces a queryable CPG without writing any C parser. Keep as Plan-B for syscall-surface extraction. |

The defensible academic framing for the judges is now: *"CodeQL Rust went GA in Oct 2025, but its query model is security-oriented; we use tree-sitter + ra-ap because structural fingerprints are first-class in the AST/HIR layer, not as taint analyses. Joern's Rust frontend is < 3 months old as of submission."*

## 4. MinHash + Jaccard on typed structural features

Drawing on **CCFinder (token-shingle k=50)**, **NiCad (line-granularity, threshold 0.7)**, **SourcererCC (bag-of-tokens, threshold 0.8 — "8 means clones will be flagged at 80% similarity" per their README)**, and the `datasketch` documentation:

**Recommended v1 config** (per fingerprint dimension, then combine):

```
num_perm = 128                 # MinHash signature length
shingle  = typed tuples        # e.g. (field_type, field_position) for TrapFrame;
                               #      (callee_name, callsite_kind) for boot-path
threshold (Jaccard)
  boot path     : 0.55         # short feature set (~10 elems); be permissive
  trap frame    : 0.85         # very structured; high bar
  scheduler     : 0.65         # medium structure
  memory mgmt   : 0.60
  file system   : 0.55
  IPC           : 0.50         # most variable across kernels
  syscall list  : 0.70

LSH bands/rows for num_perm=128:
  b=16, r=8                    # sharp probability jump near t≈0.7 — matches
                               # the typical "is this a rCore fork?" cutoff
```

Combine per-dimension Jaccards with a learned (or hand-set) weight vector — Stage 2 already says "deterministic structure, LLM only for prose"; we agree, and recommend a fixed weight vector for v1 (no learning), since we will have ≤ 40 labelled fork-pairs which is too few to fit.

## 5. Embedding / LSH for the soft-prose layer

For docstring / comment / README embeddings — *not* code:

| Model | License | Size | MTEB-multi | Reproducibility cost |
|---|---|---|---|---|
| `bge-m3` (BAAI) | MIT | 0.6 B | 59.56 | Trivial — runs on a single 8 GB GPU |
| `Qwen3-Embedding-0.6B` | Apache-2.0 | 0.6 B | ~64 | Trivial |
| `Qwen3-Embedding-8B` | Apache-2.0 | 8 B | 70.58 (top of leaderboard Jun 2025) | Needs ≥ 20 GB VRAM |
| `voyage-3` | Closed API, $0.06 / M tokens | n/a | NDCG@10 76.72 (own bench) | Cheap to call, but locks us into an API |

**Recommendation:** `bge-m3` for the v1 (open weights, well understood by the team, no per-query cost), with `Qwen3-Embedding-0.6B` as a one-line drop-in upgrade if benchmarks justify. **Avoid voyage-3** — academic reproducibility outweighs the 5-point MTEB gain, and the corpus is small (40 × maybe-30-docstrings each ≈ a few thousand vectors). The LSH index over the embeddings can reuse `datasketch` MinHashLSH after binarising (or just use `faiss` flat — 40 × N is trivially small).

## 6. Static-analysis precedents for OS-kernel structure extraction

A real survey, with what we can and cannot inherit:

- **Coccinelle (Lawall et al., INRIA).** Semantic patches for C; ships ~70 scripts inside `scripts/coccinelle/` in mainline Linux. **C-only.** We can lift their patterns for memory-allocator detection (`kmalloc`/`vmalloc`/`alloc_pages`) directly into our memory-mgmt fingerprint.
- **Glean (Meta).** First-class C/C++ indexing; Rust via SCIP/LSIF (rust-analyzer-backed). Indexes are queryable in Angle, similar to Datalog. Overkill for a 6-month student project but worth a citation as the "industrial precedent" for what we are doing in miniature.
- **Linux architectural-drift literature.** Rosik et al. ("Assessing architectural drift in commercial software development", IST 2011) used a Reflexion-model on the Linux kernel; Ahmed & Gokhale ("Linux bugs: lifecycle and architectural analysis") classifies bugs by subsystem. These give us the *vocabulary* for fingerprint dimensions ("subsystem", "drift") but no off-the-shelf extractor.
- **Gershuni et al., "Simple and precise static analysis of untrusted Linux kernel extensions"** — eBPF-specific, not directly applicable.
- **`rust-code-analysis` (Mozilla).** Tree-sitter-based, multi-language. Good template for our visitor architecture.
- **scip-rust (Sourcegraph).** Production ra-ap consumer; we should read its main.rs before we write ours.

Nobody has, to our knowledge, published a "7-fingerprint OS-kernel-structure extractor". That gap is precisely the novelty story.

## 7. Concrete v1 toolchain — buildable in one sprint

```toml
# pyproject.toml — driver layer (Python)
tree-sitter        = "0.23.*"     # bindings
tree-sitter-languages = "1.10.*"  # ships pre-built rust/c grammars
datasketch         = "1.6.*"      # MinHash + LSH
networkx           = "3.*"        # CPG-lite graph
pydantic           = "2.*"        # fingerprint.json schema
fastembed          = "0.4.*"      # bge-m3 without torch ceremony
clang              = "16.*"       # via apt; for `clang -E -P` header pre-pass
```

```toml
# Cargo.toml — semantic-Rust sidecar binary `rust-fp` (one binary, stdin/stdout JSON)
[dependencies]
ra_ap_hir            = "=0.0.286"   # pin exact; bump deliberately
ra_ap_ide            = "=0.0.286"
ra_ap_load-cargo     = "=0.0.286"
ra_ap_project-model  = "=0.0.286"
ra_ap_syntax         = "=0.0.286"
serde_json           = "1"
```

**Pipeline (per repo):**
1. `clang -E -P -nostdinc <kernel.c> > .preproc.c` — for syscall surface only.
2. Python driver walks the repo, runs tree-sitter on every `.rs` and `.c` (and `.preproc.c`), emits per-file AST JSON.
3. For Rust crates only, the driver invokes the `rust-fp` sidecar binary once per crate. Sidecar emits HIR-resolved JSON (call graph + struct layouts + type-of-each-`fn`-return).
4. A single Python `fingerprint.py` module merges both streams into the typed feature sets for the 7 dimensions.
5. `datasketch.MinHash`/`MinHashLSH` builds the similarity index.
6. `bge-m3` embeds docstrings into faiss-flat for the prose-similarity sidecar.

**Output schema (`fingerprint.json`, ~5 KB):**

```json
{
  "repo": "rCore-Tutorial-v3@2024w12",
  "boot_path":   {"entry": "_start", "hops": ["_start","rust_main","kernel_init"],
                  "csr_writes": ["satp","stvec","sscratch","sstatus"],
                  "minhash":   "..."},
  "trap_frame":  {"struct": "TrapContext",
                  "fields": [["x","[u64;32]"],["sstatus","Sstatus"],["sepc","usize"]],
                  "minhash": "..."},
  "scheduler":   {"class": "round_robin",   /* §8 below */
                  "ready_queue_kind": "VecDeque",
                  "task_struct_size_hint": 18,
                  "minhash": "..."},
  "memory_mgmt": {"allocator": ["buddy_system_allocator"],
                  "page_table_levels": 3, "asid": false, "minhash": "..."},
  "file_system": {"vfs_traits": ["File","Inode"], "fs_impls": ["easy-fs"],
                  "minhash": "..."},
  "ipc":         {"primitives": ["pipe","signal"], "shm": false, "minhash": "..."},
  "syscalls":    {"count": 23,
                  "ids": [64,93,124,172, ...], "minhash": "..."}
}
```

Acceptance for the sprint: this JSON is produced for rCore-Tutorial-v3, rCore-OS, xv6-riscv-fork, and one custom student repo in ≤ 90 s total.

## 8. One concrete sketch per fingerprint

```scheme
; 8.1 Boot path — find the entry symbol and the next 3-5 hops.
;     Rust: tree-sitter-rust query
((function_item
   (attribute_item) @attr            ; #[no_mangle] or #[link_section=".text.entry"]
   name: (identifier) @entry
   (#match? @attr "no_mangle|entry"))
 (#set! kind boot-entry))
;     C: tree-sitter-c equivalent — `_start`, `start`, `kmain`.
```

```scheme
; 8.2 Trap frame — canonicalize field order to a 33-tuple.
(struct_item
  name: (type_identifier) @s
  (#match? @s "TrapContext|TrapFrame|trapframe|trap_regs")
  body: (field_declaration_list) @fl) @S
;  → emit (s, [(field_name, type_str), ...]) and pad to 33 with NULL.
```

For **§8.3 scheduler class** — the most interesting case — the rule cascade is:

```
Inputs from tree-sitter + ra-ap:
  R = set of fn names containing /schedule|switch|pick_next|tick/
  Q = the data structure type the scheduler dequeues from (resolved by ra-ap)
  P = priority field present on the task struct? (bool, integer width)
  T = is there a "vruntime"-like monotonically increasing key? (RB-tree usage)
  L = number of priority levels (count of arrays in ready queues)

Classifier:
  if T and uses BTreeMap/RBTree            → "CFS-like"
  elif L >= 3 and feedback-on-blocked       → "MLFQ"
  elif L == 0 and Q in {VecDeque, LinkedList} and !P
                                            → "round-robin"
  elif P and L == 0                         → "priority-FIFO"
  elif ad-hoc                               → "custom" (LLM prose explains)
```

Tree-sitter query to find ready queues:

```scheme
(struct_item
  name: (type_identifier) @scheduler_struct
  (#match? @scheduler_struct ".*[Ss]cheduler.*|TaskManager")
  body: (field_declaration_list
          (field_declaration
            name: (field_identifier) @field
            type: (_) @ty
            (#match? @field "ready|queue|run_queue|rq"))))
```

The remaining four are similarly mechanical:

- **§8.4 Memory mgmt** — query for `#[global_allocator]` annotation (Rust) or `kmalloc_caches`/`buddy_*` symbols (C); read page-table struct depth from the type chain `PageTable → PageTableEntry`; ASID flag = presence of an `asid` field in the address-space struct.
- **§8.5 File system** — collect every `impl File for X` and `impl Inode for X` in Rust; in C, every struct ending in `_operations` with a `.read`/`.write` member. The fingerprint is the multiset of trait names.
- **§8.6 IPC** — token-set over the source for `{pipe,signal,futex,mqueue,shm,socket,channel}` *plus* the set of trait/impl pairs touching them. Cheap, high recall, low precision — acceptable here because the LLM writes the prose.
- **§8.7 Syscalls** — for Rust, parse the giant `match syscall_id { ... }` (rCore convention) into `(id, handler_name)` pairs; for C, walk every `SYSCALL_DEFINEn(name, ...)`. Both emit `(canonical_name, arity, return_type)` tuples; MinHash over that set.

---

**Bottom line.** The Stage 2 stack — tree-sitter + ra-ap + custom CPG-lite, MinHash/Jaccard for structural similarity, bge-m3 for soft prose — survives, with **one correction**: the *justification* for skipping CodeQL/Joern must be updated. CodeQL Rust is GA (Oct 2025), Joern has a rust2cpg frontend (Apr 2026); we still skip both, but for *reproducibility* and *maturity* reasons rather than "no Rust support". The seven-fingerprint scheme above is implementable in one sprint by two people with the toolchain pinned exactly as shown.
