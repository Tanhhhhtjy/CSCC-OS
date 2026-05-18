# Stage 3 / Document D — Fingerprint Extraction Spec and "精准无误" Grading Operationalization

**Project:** proj18 — Small-OS analysis & comparison agent
**Status:** Buildable spec, not aspirational. Every claim below maps to a verified upstream artifact or a buildable extractor.
**Scope:** Part 1 — extractor spec for the 7 fingerprint dimensions. Part 2 — operational definition of "精准无误". Part 3 — differentiation rehearsal against KFC-agent (2025 first prize).

---

## Part 1 — Per-dimension extraction spec

Each dimension follows the same contract:
1. **Ground-truth artifact** — the file(s) that *are* the answer; everything else is inference.
2. **Canonical normalization** — typed schema the fingerprint compares against.
3. **Cross-repo equivalence rule** — when do an rCore artifact and an xv6 artifact count as "the same dimension"?
4. **Disambiguation / hard cases.**

### 1.1 Boot path

**Ground-truth artifact.** The linker entry symbol plus the first three call edges.
- rCore-Tutorial-v3: `os/src/entry.asm` — `_start: la sp, boot_stack_top; call rust_main` (verified upstream). Edge 1 = `_start → rust_main`; edge 2 = inside `os/src/main.rs::rust_main` → `clear_bss` and `trap::init`; edge 3 = `trap::init → set_kernel_trap_entry`.
- xv6-riscv: `kernel/entry.S` — `_entry: ... call start`; edge 2 = `start → main`; edge 3 = `main → ...` over `kinit/kvminit/procinit/...`.
- ArceOS: `modules/axhal/src/platform/<plat>/boot.rs` — `_start` written in `#[naked]` Rust; edge 1 = `_start → rust_entry`; edge 2 = `rust_entry → axruntime::rust_main`.
- Asterinas: framekernel boot lives under `ostd/` (the confined-unsafe trusted base) → `kernel/` entry. The framekernel split itself is part of the fingerprint.

**Canonical normalization.** Directed acyclic graph fragment with ≤ 8 nodes:
```json
{"entry": "_start", "edges": [["_start","rust_main","asm→rust"], ...],
 "asm_lines": 12, "rust_lines": 0, "first_high_level": "rust_main"}
```
Plus a scalar `asm_rust_ratio` (asm_lines / (asm_lines + rust_lines) along the first-three-edges path).

**Cross-repo equivalence rule.** Two boot paths are "the same family" iff (a) same `first_high_level` symbol (after demangling), and (b) `asm_rust_ratio` within ±0.1. A 2022 submission that lifts rCore's `entry.asm` verbatim will hit ratio = 1.0 at edge 0 + symbol `rust_main` and match at family level. A team that rewrote boot in pure Rust (`_start: #[naked] fn`) will diverge here.

**Hard cases.**
- LinkerScript-only differences (some teams move `_start` symbol via `ENTRY(_start_smp)` in `linker.ld`). Rule: parse `linker.ld` first, take `ENTRY()` as the truth, fall back to `.global _start` scan only if no `ENTRY()` directive.
- SBI-launched kernels jumping into `M-mode` shim before `_start`. Rule: ignore pre-`_start` SBI; the corpus is uniformly OpenSBI-fronted and SBI is not the team's code.
- Submissions that conditionally compile a multi-arch boot (RISC-V + LA64 + AArch64 — Starry-OS does this). Rule: produce one boot-path fingerprint per arch present in `Cargo.toml [target.'cfg(...)']` features; cross-repo equivalence is computed per-arch.

### 1.2 Trap frame layout

**Ground-truth artifact.** The struct definition closest to the trap entry symbol that is written to from assembly.
- rCore-Tutorial-v3, `os/src/trap/context.rs`:
  ```rust
  #[repr(C)] pub struct TrapContext {
      pub x: [usize; 32], pub sstatus: Sstatus, pub sepc: usize,
      pub kernel_satp: usize, pub kernel_sp: usize, pub trap_handler: usize,
  }
  ```
  (verified)
- xv6-riscv, `kernel/proc.h`: 35 ordered `uint64` fields with byte offsets in comments: `kernel_satp@0, kernel_sp@8, kernel_trap@16, epc@24, kernel_hartid@32, ra@40, sp@48, gp, tp, t0..t6, s0..s11, a0..a7`. (verified — full ordering captured.)
- ArceOS: `TrapFrame` in `modules/axhal/src/arch/riscv64/context.rs` — register array `regs: [usize; 31]` (omits `x0`) + `sepc` + `sstatus`. Layout matches the ABI but omits the kernel-side fields rCore appends.
- Asterinas: `ostd/src/arch/x86_64/trap/...` — x86-64 trap frame (rax/rcx/.../rip/cs/rflags/...); not directly comparable to RISC-V frames; comparison flags the *architecture* axis first.

**Canonical normalization.** Ordered list of triples:
```
[(field_name, width_bytes, type_class), ...]
```
where `type_class ∈ {gpr, csr, ptr, scalar, padding}`. From this we compute:
- `total_size_bytes` (sum)
- `n_gpr` (count of `gpr`)
- `kernel_fields` (count of fields named matching `kernel_*` or `trap_handler`)
- A *position-sensitive MinHash* over the tuple sequence (NOT a set MinHash — order is load-bearing because trap-entry assembly indexes by offset).

**Cross-repo equivalence rule.** Two frames are "the same" iff (a) same architecture (RISC-V64 vs x86-64 vs AArch64 — coarse axis), (b) MinHash Jaccard ≥ 0.8 on the field sequence, (c) `kernel_fields` differs by ≤ 1. Rule (c) is what separates rCore-derivatives (3 kernel fields: `kernel_satp`, `kernel_sp`, `trap_handler`) from xv6-derivatives (3 different kernel fields: `kernel_satp`, `kernel_sp`, `kernel_trap`) — both have 3 kernel fields, MinHash will likely be < 0.8, and the dimension correctly distinguishes them. Confirmed against upstream data above.

**Hard cases.**
- Multiple `TrapContext`-like structs (one for U→S, one for S→S). Rule: pick the one referenced from the symbol that the IDT/`stvec` points to. Fall back to the one closer in tokens to the `__alltraps` / `trap_handler` asm.
- Submissions that rename fields (`pub user_x: [usize; 32]`). Normalize field names to a 17-token canonical alphabet (`x_reg`, `sstatus_csr`, `sepc_pc`, `kernel_ptr`, etc.) before MinHash.

### 1.3 Scheduler class

**Ground-truth artifact.** The function reached from the timer-interrupt handler that picks the next runnable task. Static analysis path: `stvec → trap_handler → SupervisorTimer arm → schedule()` or equivalent.

- xv6-riscv (`kernel/proc.c::scheduler`): linear scan over `proc[NPROC]` array, picks first `RUNNABLE`. Classified `round-robin` (no priorities, no timeslice accounting — verified above).
- rCore ch5+ (`os/src/task/manager.rs::TaskManager::fetch`): `VecDeque::pop_front` + `push_back` on yield. Classified `round-robin` (FIFO queue).
- ArceOS (`modules/axtask` + the `scheduler` crate): exports `FifoScheduler`, `RRScheduler`, `CFScheduler` (verified upstream). Active class is chosen by Cargo feature: `--features sched_fifo` / `sched_rr` / `sched_cfs`.
- Starry-OS: monolithic OS layered on ArceOS, inherits ArceOS scheduler choice — fingerprint records the feature flag actually compiled in.

**Canonical normalization.**
```json
{"class": "round_robin|mlfq|cfs|priority|fifo|custom",
 "evidence_file": "os/src/task/manager.rs", "evidence_line": 47,
 "structural_signals": {"has_runqueue_deque": true, "has_priority_field": false,
                        "has_vruntime": false, "uses_timeslice_counter": false}}
```

**Classification rules (AST-shape based, no LLM in the loop):**
- `has_vruntime` (Rust: struct field named `vruntime` OR `v_runtime`; C: `vruntime`) → CFS-like.
- `has_priority_field` AND ≥ 2 distinct queues / array-of-deque → MLFQ.
- `has_priority_field` AND single queue with comparator → priority.
- single FIFO queue, no priority, no timeslice → round-robin.
- timeslice present but no priority → round-robin with quantum.
- none of the above → `custom`, manual-review flag.

**Cross-repo equivalence rule.** Same `class` token = equivalent at coarse granularity. At fine granularity, structural-signals dict Jaccard. Two CFS-likes are "really equivalent" only if `has_vruntime ∧ uses_timeslice_counter`.

**Hard cases.**
- **Comment-vs-code disagreement.** A submission with `// MLFQ scheduler` in the docstring but a single FIFO `VecDeque` underneath. The structural rule wins; the report says "claimed MLFQ in comment, implemented as round-robin (evidence: `task/manager.rs:47` — single `VecDeque<Arc<TaskControlBlock>>`)." This case is in the red-team set (§ Part 2).
- **Scheduler in an external crate.** If `Cargo.toml` pulls `scheduler = { ... }`, the extractor walks into the vendored crate. If the team used `crates.io` upstream, the fingerprint records `external:scheduler::FifoScheduler` and the comparison is at the dependency-version level.

### 1.4 Memory management

**Ground-truth artifact.** Three sub-facts, each with its own evidence:
1. **Page-table walk depth**: grep for the page-table mode constant.
   - `SATP_MODE_SV39 = 8 << 60` or `Sv39` enum variant → 3-level. rCore-Tutorial-v3 default.
   - `Sv48` → 4-level (some 2024 submissions).
   - `Sv57` (rare).
2. **Allocator class**: `Cargo.toml` dependency analysis + `#[global_allocator]` symbol.
   - `buddy_system_allocator` crate → buddy + linked-list combo. rCore default.
   - `slab_allocator` → slab.
   - `linked_list_allocator` → bump+free-list.
   - hand-rolled (no `[global_allocator]` from a known crate) → `custom`.
3. **CoW present**: existence of a `cow_*` function family AND a `Frame::ref_count` / `Arc<PhysPageNum>` reference-counted physical page.

**Canonical normalization.**
```json
{"pt_levels": 3, "pt_mode": "Sv39", "allocator": "buddy_system_allocator@0.10",
 "cow": false, "evidence": {...per-fact file:line...}}
```

**Cross-repo equivalence rule.** Identical `(pt_mode, allocator_crate, cow)` triple = equivalent. Otherwise per-dimension Jaccard.

**Hard cases.**
- Submission imports `buddy_system_allocator` but routes large allocs through a custom path. Rule: the *registered* `#[global_allocator]` is the truth; sidecar allocators are listed under `auxiliary_allocators` field and not used for equivalence.
- Multi-arch repos with different page-table modes per arch — record per-arch, equivalence per-arch.

### 1.5 File system

**Ground-truth artifact.** SuperBlock struct + magic number constant.

- rCore easy-fs (`easy-fs/src/layout.rs`): `SuperBlock { magic: u32, total_blocks: u32, inode_bitmap_blocks: u32, inode_area_blocks: u32, data_bitmap_blocks: u32, data_area_blocks: u32 }` and `DiskInode { size, direct[INODE_DIRECT_COUNT], indirect1, indirect2, type_ }` (verified). Magic constant in same file `EFS_MAGIC = 0x3b800001`.
- FAT32 ports: BPB signature `0x55AA` at offset 510 + `FAT32` string at offset 82 in boot sector. Detected by literal-byte search on `mkfs` output or by struct field names `bpb_*`.
- ext2/ext4 ports (rare in this corpus): magic `0xEF53`.
- ByteOS / from-scratch FS: classify as `custom`, record superblock field list for novelty scoring.

**Canonical normalization.**
```json
{"fs_class": "easy_fs|fat32|ext2|custom",
 "magic": "0x3b800001", "superblock_fields": [...ordered...],
 "inode_direct_count": 28, "has_indirect2": true}
```

**Cross-repo equivalence rule.** Same `fs_class` AND same `magic` → identical family. easy-fs forks that change `INODE_DIRECT_COUNT` from 28 to some other value are flagged as `easy_fs (modified)` and the constant difference goes into the diff narrative.

**Hard cases.**
- A team vendored easy-fs but added journaling on top. Rule: the inner superblock is still `easy_fs`; the journaling layer is recorded under `fs_extensions: ["journal"]`.
- VirtIO-block-only with no on-disk FS (early-stage submissions). Record `fs_class: "none", block_device_only: true`.

### 1.6 IPC

**Ground-truth artifact.** Presence/shape of four IPC primitives plus the syscall numbers that bind them:
- **Pipe**: `Pipe` struct + ring buffer field + matching `SYSCALL_PIPE` (= 59 in rCore, verified).
- **Signal**: `SignalFlags` bitset + `sys_kill` handler. rCore-Tutorial-v3 has `SYSCALL_KILL = 129` (verified).
- **Mutex/semaphore/condvar**: rCore exposes `SYSCALL_MUTEX_CREATE = 1010 … SYSCALL_CONDVAR_WAIT = 1032` (verified — non-Linux range, telltale rCore fingerprint).
- **Shared memory / futex / mailbox**: Linux-style `futex` (98) appears in CSCC submissions trying to pass musl libc tests; presence flips a `futex_present` bool.

**Canonical normalization.**
```json
{"primitives": {"pipe": true, "signal": true, "mutex": "userspace_id_table",
                "semaphore": true, "condvar": true, "futex": false,
                "mailbox": false, "shmem": false},
 "syscall_binding": {"SYSCALL_PIPE": 59, "SYSCALL_MUTEX_CREATE": 1010, ...}}
```

**Cross-repo equivalence rule.** Boolean-vector Jaccard over the primitives dict (length 8). Tie-broken by syscall-number Jaccard — two repos that both implement `mutex` but one binds it to `1010` (rCore native) and the other to `98` (Linux `futex`) are different at the syscall-binding axis even if equivalent at the primitives axis.

**Hard cases.**
- "Stub" syscalls — handler exists but body is `unimplemented!()` or `return -ENOSYS`. The extractor must classify these as *advertised but unimplemented*. Rule: walk the AST of the handler; if it has fewer than 3 statements and one of them is the literal `unimplemented!()`/`panic!()` macro or a return-of-negative-constant, mark `stub: true`.

### 1.7 Syscall surface

**Ground-truth artifact.** The `SYSCALL_*` const block (rCore-style, `os/src/syscall/mod.rs`) or `syscalls.h` (xv6-style) plus the `match`/`switch` dispatch table in the syscall handler.

For rCore-Tutorial-v3 we verified 35+ constants spanning 24…3001 (DUP, OPEN, CLOSE, PIPE, READ, WRITE, EXIT, SLEEP, YIELD, KILL, GET_TIME, GETPID, FORK, EXEC, WAITPID, THREAD_CREATE, GETTID, WAITTID, MUTEX_*, SEMAPHORE_*, CONDVAR_*, FRAMEBUFFER_*, EVENT_GET, KEY_PRESSED).

**Canonical normalization.**
```json
{"syscalls": [{"name": "sys_open", "number": 56, "linux_equivalent": 56,
               "handler_file": "os/src/syscall/fs.rs", "handler_line": 41,
               "is_stub": false, "argc": 3}, ...],
 "total_count": 35, "linux_overlap": 18, "rcore_specific": 13}
```

`linux_equivalent` is populated from a static map (Linux RISC-V syscall numbers — they're stable). Anything outside the standard table gets `linux_equivalent: null`. The fingerprint summary is then:
- `total_count`
- `linux_overlap` (count of syscall numbers that match Linux RISC-V)
- `rcore_specific` (numbers in the 1000-3001 rCore-native range)
- `stub_count`

**Cross-repo equivalence rule.** Three-axis Jaccard: (syscall-name set), (syscall-number set), (handler-signature set). The third matters because two repos can both expose `sys_open` but one takes `(path, flags)` while the other takes `(dirfd, path, flags, mode)` — that's a Linux-compat vs rCore-native distinction worth surfacing.

**Hard cases.**
- Generated syscall tables (macro-expanded). Rule: run `cargo expand` first if the const block is empty after tree-sitter and the file contains a `syscall_table!` macro invocation.
- Conditional compilation (`#[cfg(feature = "musl")]`). Record per-feature-set; the manifest captures which feature was actually enabled in the team's build.

---

## Part 2 — Operationalizing "精准无误"

### 2.1 What is a "fact"?

Inside any Describe-Agent or Compare-Agent output, exactly four fact classes exist:

| Class | Example | Verifiability |
|---|---|---|
| **F-CITE** | "`os/src/trap/context.rs:8` defines `TrapContext`" | grep against repo + line-existence check |
| **F-NUM** | "the trap frame has 35 ordered fields" | round-trip against extractor output |
| **F-STRUCT** | "the scheduler is round-robin" | must equal `fingerprint.json.scheduler.class` |
| **F-CAT** | "this scheduler shows simple fairness semantics" | prose adjective; no verification, but cap on prevalence |

Everything in the report is one of these four. There is no fifth class. If the LLM emits a sentence that doesn't decompose into F-CITE / F-NUM / F-STRUCT / F-CAT tokens, the post-validator drops the sentence.

### 2.2 Verification mechanisms

**F-CITE — post-hoc tool, hard gate.**
- Regex out every `(path:line)` and every `` `path` `` backtick-quoted path from the report.
- `git ls-tree` to confirm path exists; `wc -l` to confirm line number ≤ file length; AST-touch to confirm a symbol exists at that line (function/struct/const).
- **Any failure → fact rewritten as "evidence unavailable" or the sentence is removed.** No exceptions, no "the LLM probably meant…" inference.

**F-NUM — round-trip oracle, hard gate.**
- Every numeric claim is paired with a JSON-path into `fingerprint.json` at decoding time via a structured-output schema (Anthropic tool-use response, or constrained decoding).
- Validator runs `assert report_number == fingerprint[json_path]`; mismatch → fact rewritten to the extractor value or sentence dropped.

**F-STRUCT — extractor is the oracle, hard gate.**
- The LLM never invents structural categories. The Describe-Agent receives a *menu* of legal tokens (`{round_robin, mlfq, cfs, priority, fifo, custom}`) and must select; the prompt forbids freeform synonyms ("FIFO-style", "RR-like"). Post-validator does a strict-set membership check.

**F-CAT — soft cap.**
- Adjective sentences allowed up to **≤ 15% of report tokens**, measured by a small classifier (or simple part-of-speech / adjective-density heuristic).
- The prompt explicitly states: "No comparative adjective without a F-NUM or F-CITE in the same sentence."

### 2.3 Fact taxonomy with priority tiers

**P0 — must-verify-before-ship (blocker if any failure).** F-CITE, F-STRUCT.
- One bad citation in a randomly chosen report = brief-defined B-grade exit. Therefore P0 is a *publish-time* gate, run on every report before it is rendered in the dashboard.
- Implementation: a `validate_report.py` step in the Sprint-5 pipeline. Exit non-zero → CI fails → report is not shipped.

**P1 — round-trip-required (warn if failure, can ship with annotation).** F-NUM.
- Numeric drift between report and fingerprint is almost always an LLM rounding artifact. Mismatch is auto-rewritten to the extractor value; the rewriter logs the delta so we can see how often the LLM "wandered".

**P2 — adjective cap (lint-level).** F-CAT.
- Soft warning if adjective density > 15%. Doesn't block ship, but the Sprint-5 spot-check rubric counts dense-adjective reports against us.

### 2.4 Red-team test set (20 cases)

Built before Sprint 5, run as a regression suite every commit. The system must classify each correctly. Format: `<scenario> → expected fingerprint behavior`.

1. Repo with `// Implements MLFQ` comment but single `VecDeque` runqueue → `scheduler.class = round_robin`, comment ignored.
2. Repo that names its trap frame `struct UserContext` (not `TrapContext`) → still classified as trap frame via stvec→handler→struct chase.
3. Repo with two `TrapContext` definitions (U-mode and S-mode) → picks the one referenced from stvec.
4. Repo that vendored `buddy_system_allocator` and renamed it `my_allocator` → allocator classified by source-content hash, not crate name.
5. Repo whose `Cargo.toml` declares `slab_allocator` but never registers it as `#[global_allocator]` → allocator classified as `linked_list_allocator` (the one actually registered).
6. Repo claiming "ext4 support" in README but with easy-fs superblock magic `0x3b800001` → `fs_class = easy_fs`, README claim ignored.
7. Repo with all 200 Linux syscalls declared as constants but 180 stubbed (return `-ENOSYS`) → `total_count = 200, stub_count = 180`. The compare-agent prose must surface the stub ratio.
8. Repo with macro-generated syscall table (`syscall_table!`) → fall back to `cargo expand`, extract from expansion.
9. Repo that wraps rCore-Tutorial-v3 ch6 verbatim with `git subtree` → MinHash similarity ≥ 0.95 across all 7 dimensions; novelty score must report this honestly.
10. Repo that forked rCore but rewrote the trap frame in pure-RISC-V assembly with no Rust struct → trap-frame extractor degrades gracefully to "asm-defined, hand-review required", does not fabricate fields.
11. Repo using ArceOS as a dependency, scheduler chosen by `--features sched_rr` → fingerprint records `external:scheduler::RRScheduler`, not `custom`.
12. Repo using ArceOS but with a locally-patched `axtask/run_queue.rs` → fingerprint records `external:scheduler (patched)`, with diff hash.
13. Repo using LoongArch instead of RISC-V → trap-frame extractor must handle the LA64 CSR set (`crmd`/`prmd`/`era`) without crashing.
14. Repo with x86-64 + RISC-V dual build (Asterinas-style framekernel) → produces per-arch fingerprint, equivalence is per-arch.
15. Repo with no scheduler (single-task kernel) → `scheduler.class = "none", n_tasks = 1`; not `custom`.
16. Repo whose `_start` is defined in a separate crate pulled via path-dep → boot-path walker follows path-deps, doesn't stop at the crate boundary.
17. Repo with a `TrapContext` definition copy-pasted from rCore but with one field renamed (`sstatus → status_csr`) → MinHash on canonical-token-normalized fields still ≥ 0.95 vs. rCore.
18. Repo with deliberately misleading file paths (`os/src/scheduler/mlfq.rs` containing round-robin code) → classification by AST shape, file-name ignored.
19. Repo with a CoW implementation that's commented out (`#[cfg(feature = "cow")]` and the feature is off) → `cow: false` for the active build; an `available_features` field records that cow exists but isn't enabled.
20. Repo where the LLM is most likely to hallucinate — a single-file 200-line kernel (e.g., a teaching toy submitted by mistake). The Describe-Agent must produce a short report with all 7 dimensions either filled or honestly marked `"unknown — kernel too small"`, never fabricated.

The red-team set must pass at ≥ 18/20 before any human spot-check. Failures on 9, 17, 18 in particular are *expected* on the first run and are how we tune the equivalence thresholds.

---

## Part 3 — Differentiation rehearsal: "what makes this not just another LLM-agent project?"

The 2025 first-prize KFC-agent for kdump analysis is the comparison the evaluator will silently make. Its architecture, as widely shared in the OS-track grapevine: a Planner + Memory + Bash-tool ReAct loop wrapped around a Linux kdump corpus, with the LLM driving `gdb`/`crash`/`addr2line` over crash dumps and producing a root-cause report. That's an excellent project for a different problem. Our team's answer to the inevitable question must hit three points, in order:

**(1) Different oracle, different failure mode.** KFC-agent's oracle is the running `crash` tool: the LLM asks `crash` a question, `crash` returns ground-truth bytes. Our oracle for the seven structural dimensions is **a tree-sitter + AST extractor that runs without an LLM in the loop**. The LLM never decides "is this scheduler round-robin?" — the AST rule decides, and the LLM writes the prose around the decision. This is not a "stylistic preference"; it is the only way to honor the brief's "精准无误" demand. KFC-agent can afford LLM-in-the-loop because `crash`'s output is itself ground truth; we cannot, because once the LLM is allowed to classify "this is MLFQ", there's no recovery path to "actually it was round-robin all along — the comment lied".

The killer slide: **side-by-side of the same repo (red-team case #1) processed by a naive LLM-agent (which reads the comment, says "MLFQ") versus our pipeline (which reads the AST, says "round_robin; comment claims MLFQ, structurally false").** This single slide is worth more than 30 minutes of architecture diagrams.

**(2) Structural fingerprints are a typed artifact, not a chat transcript.** Every report we ship is reproducible from `fingerprint.json` + git SHA. A reviewer can:
- run `make fingerprint corpus/2022/Byte_OS` themselves,
- diff the produced JSON against the one we shipped,
- get byte-identical output (the LLM-prose portion is the only non-deterministic slice, and it's bounded to F-CAT-class adjectives).

KFC-agent's transcript is not reproducible — different LLM session, different chain-of-thought. **Reproducibility is the second killer demo: re-run the whole 40-repo corpus live during judging, in under 3 minutes, deterministic output.** A judge cannot get this from a Planner+Memory agent.

**(3) Domain-specific normalization that a general agent cannot acquire.** The seven dimensions are not arbitrary — they are precisely the dimensions that distinguish CSCC submissions from each other (boot, trap, sched, MM, FS, IPC, syscall — verified across rCore-Tutorial-v3, xv6-riscv, ArceOS, Asterinas, Starry-OS, all of which have their own non-overlapping idioms in each dimension, as Part 1 documents in concrete detail). A judge who watched KFC-agent last year will recognize this immediately: the prior winner had one specialized oracle (`crash`). We have **seven specialized oracles**, each grounded in real upstream artifacts (verified file paths and field lists for each fingerprint), each defended against fabrication by a hard-gate validator.

**The two-minute team script for the Q&A "isn't this just RAG?" challenge:**

> "RAG retrieves chunks and asks the LLM to reason over them. We retrieve **typed structural artifacts** — a 35-field trap frame, a 35-entry syscall table, a 6-field superblock — and ask the LLM only to write narrative paragraphs *around* facts the extractor has already locked. The LLM is downstream of accuracy, not responsible for it. If you randomly pick a report and grep one of its file:line citations against the repo, the path will exist and the line will be where we said it is — because we run that grep ourselves on every report before we render it, and we drop any sentence that fails. RAG cannot make that promise; that's the difference."

The third demo, alongside (1) and (2): **the post-hoc citation validator running live on the dashboard.** Every cited path turns green as the validator confirms it. A judge can click on any citation to jump to the GitHub line. This makes the "精准无误" claim auditable in real time — and it is the visual that an experienced judge will use to short-circuit the "is this another LLM gloss" suspicion.

**Three demos to anchor the differentiation:**
1. Comment-vs-code disagreement (red-team #1): structural truth beats prose.
2. Live 40-repo reproducibility run (≤ 3 min): determinism beats Planner+Memory.
3. Real-time citation validator on the dashboard: auditable accuracy beats trust.

If we have time for only one, ship #1. It is the most legible to a non-OS judge and the most damaging to the "ChatGPT-over-a-repo" framing.

---

**Bottom line on "精准无误":** This is not aspirational. It is enforced by (a) a hard-gate post-validator on every F-CITE and F-STRUCT, (b) a round-trip oracle on every F-NUM, (c) a structured-output schema that prevents the LLM from emitting free-form structural verdicts, (d) a 20-case red-team regression suite that must pass before any human review. If any of these four mechanisms slips during Sprint 4-5 development, the project leadership should kill the dashboard polish and rebuild the validator instead. The validator is the project; everything else is presentation.
