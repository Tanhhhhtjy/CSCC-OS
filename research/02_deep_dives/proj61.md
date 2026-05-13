# proj61 Feasibility Report — Agent-OS Kernel Extension

**Track:** 2026 CSCC OS Functional Challenge (B-level, engineering)
**Base:** rCore-Tutorial-v3 on RISC-V64 / QEMU, RustSBI
**Tutor:** Zhao Xia (BTBU)
**Stage 1 score:** 25/25

---

## 1. Technical path

We commit to **rCore-Tutorial-v3 `main` branch as the upstream base**, *not* uCore. Three reasons drive this: (a) rCore-v3 ships a clean chapter-7 IPC layer and chapter-4 address-space code we can reuse verbatim; (b) `riscv64gc-unknown-none-elf` + RustSBI is the lowest-friction toolchain in 2026 — the latest tutorial commit on `main` still builds cleanly with current nightly Rust + `cargo-binutils 0.3.3` + `llvm-tools-preview`; (c) keeping the kernel in safe-ish Rust lets us borrow the "framekernel" idea from Asterinas (small auditable unsafe core, the rest in safe Rust) for the new Agent subsystem without rewriting the world. uCore-C exists as a fallback if a teammate is C-only, but the Rust track wins on both productivity and "code quality" points (25% of grading).

The architectural spine is a five-layer cake added to rCore:

1. **Agent-PCB extension (`AgentTask`)** — a struct grafted onto `TaskControlBlockInner` carrying `agent_type`, `heartbeat_interval`, `resource_quota`, `loop_state`, and `context_path_meta` (head/tail/len/quota). Lives entirely in kernel space; never directly readable from user.
2. **Agent Context Region (ACR)** — a contiguous user-VA range mapped at fork/spawn time between user stack base and heap top, backed by physical frames the kernel owns and can reclaim. Two zones: a fixed-size *control header* (cursor offsets, ring metadata) and a quota-bounded *content ring* (path nodes). Layout deliberately mirrors a single-producer-single-consumer ring so user-mode reads need zero syscalls.
3. **Structured tool-call ABI (`sys_tool_call`)** — *not* JSON. We use a **MCP-shaped binary frame**: 4-byte magic + u16 tool_id + u16 nparams + (key_len, key_bytes, val_type, val_len, val_bytes)* + reserved. Tool IDs are issued by `sys_tool_list`. JSON-RPC framing is the public wire-level inspiration (MCP itself runs JSON-RPC 2.0 over stdio/HTTP per the 2025-11-25 spec), but inside the kernel we deserialize to a fixed C-repr struct because (i) bringing `serde_json` into a `no_std` kernel costs ~120KB and (ii) parsing untrusted JSON in ring 0 is a security smell. We document the JSON↔TLV bijection so an external MCP client could be bridged in user space.
4. **Context-path subsystem** — append-only ring with FIFO + an LRU "hot tag" promotion bit. `sys_context_push` writes the *summary* (≤256 bytes) into the kernel-controlled metadata slot AND lets the kernel `copy_to_user` the body into the user-mapped ACR. `sys_context_query` is mostly a no-op: user code reads ACR directly. `sys_context_rollback(n)` rewinds `tail` and zeroes the orphaned slots.
5. **Agent Loop scheduler hook** — a new task state `TaskStatus::AgentBlocked`. `sys_agent_wait` deschedules with a wakeup mask `{HEARTBEAT, IPC_MSG, FILE_EVENT}`. Heartbeats piggy-back on the existing timer IRQ path; IPC/file events are posted by other syscall handlers via an in-kernel event bus (MPSC ring per Agent task).

For the file system, we extend `easy-fs` (the tutorial's `EasyFileSystem`) with a sidecar **attribute table**: a single inode-indexed open-addressing hash file storing `{inode_id → [(key, value)]}` plus a pre-built reverse index for the three "hot" keys (`type`, `owner`, `tag`). This avoids touching the disk layout of the existing FS (good for code-quality grading) and gives the demanded query-vs-traversal speedup.

Multi-Agent IPC piggy-backs on rCore chapter-7 mailboxes but adds a *typed envelope* with sender PID, conversation ID, and a content-block discriminator (text/tool_result/event), again deliberately structured to map 1:1 onto MCP content blocks if we ever need to bridge upward.

## 2. Four-month milestone plan (8 sprints × 2 weeks)

**Sprint 1 (W1-2) — Baseline & instrumentation.** Fork rCore-Tutorial-v3 at the latest `main` SHA, get all six chapters' tests green in QEMU, set up CI with `qemu-system-riscv64` headless + `expect` harness, write a `kerneldoc` mkdocs skeleton, draw the v0 architecture diagram in Mermaid.

**Sprint 2 (W3-4) — AgentTask & sys_agent_create.** Add `AgentTask` fields to `TaskControlBlockInner`, wire `sys_agent_create` / `sys_agent_info`. Map a 64KB ACR at user VA `0x80_0000_0000`. Acceptance: `make test-agent-spawn` spawns 10 agents in parallel, all show up in `sys_agent_info`, no leaked frames per `MemorySet` audit.

**Sprint 3 (W5-6) — sys_tool_call stub + integration test passing in QEMU.** TLV codec in `kernel/src/agent/abi.rs`, register three tools (`query_process`, `get_system_status`, `read_context`), unit-test the codec with `proptest` in `no_std`. End-to-end: user `agent_demo` issues 100 `sys_tool_call`s, dumps results, host-side `expect` script diffs.

**Sprint 4 (W7-8) — Context path + ACR ring.** Implement `sys_context_push/query/rollback/clear`, FIFO eviction, quota enforcement, ACR ring header. Run a 10k-step soak test (must not OOM, must not panic). This sprint is the riskiest for memory bugs — budget 3 days for `miri`-on-host of the codec + a manual `unsafe` audit.

**Sprint 5 (W9-10) — easy-fs attribute extension.** Sidecar attribute file + hash index, `query_file` tool, `set_attr` / `del_attr` syscalls. Bench: 1000 files, query by `tag` — must beat path traversal by ≥10× wall-clock in QEMU (target ≥50×).

**Sprint 6 (W11-12) — Agent Loop scheduler.** `TaskStatus::AgentBlocked`, heartbeat, event bus, `sys_agent_watch/wait/unwatch`. Acceptance: 4 concurrent agents, idle CPU% in QEMU monitor <5% when no events; jitter on heartbeat ≤2 ticks @ 100Hz.

**Sprint 7 (W13-14) — Integrated demo: Agent System Administrator (Scenario B).** One janitor-agent + three worker-agents. Janitor uses `query_process`, `query_file` (by `tag=tmp`), `send_message` to ask workers to clean. Path history recorded. This is the "wow" demo — must run end-to-end unattended.

**Sprint 8 (W15-16) — Hardening, docs, video.** Run AddressSanitizer-style audit via `kani` on the unsafe blocks, write the architecture doc with three Mermaid diagrams (process layout / tool-call dataflow / Agent Loop state machine), record a 6-min screencast, freeze the submission tag `v1.0-cscc`.

The base tasks (一/二/三, 40% of grade) close by Sprint 4. The advanced tasks (四/五, 35%) close by Sprint 6. Sprints 7-8 are pure grade-maximization on the open-ended 25%.

## 3. Skills inventory & gap analysis

**Have on the team:** Rust (intermediate ≥2 members), RISC-V assembly basics, QEMU debugging, Git/CI, Mermaid. **Light-but-recoverable gaps:** `easy-fs` internals (one focused week, the rCore tutorial book chapter 6 is enough), `unsafe` Rust in `no_std` kernel context (mitigated by keeping unsafe blocks small + a `miri`-on-host harness). **Hard skills to acquire:** lock-free ring buffer design for the ACR (one teammate reads "Rust Atomics and Locks" weeks 1-2 in parallel with Sprint 1).

The existing skills we will lean on: **`superpowers:test-driven-development`** for the syscall TLV codec and the context-path ring (both have crisp invariants); **`superpowers:systematic-debugging`** for the inevitable QEMU/RustSBI mismatches; **`frontend-design:frontend-design`** for the final demo dashboard (a thin TUI in `ratatui` showing live agent loop states beats a slide-deck). **`paper-orchestra`** is overkill for a 35-page design doc — we'll lift only its outline-agent for the report skeleton.

**New skills worth authoring as part of this project:**
- `agent-runtime-spec` — codifies the TLV-wire/JSON-wire bijection and tool-call versioning rules so other teams in the lab can plug in. Reusable across proj61 *and* any future MCP-on-bare-metal work.
- `kernel-syscall-table-extender` — rCore-specific recipe: where to register a new syscall ID, where the user-side libc wrapper lives, what `make` targets to touch. Saves every new contributor a half-day.

## 4. Demo + evaluation plan

**Demo deliverable:** a 6-minute scripted QEMU session. Boot → spawn 4 agents → janitor agent enters loop → emits a `query_process` (TLV frame visible in a side terminal via a debug ringbuf) → discovers a stale tmp file via `query_file(tag=tmp)` → messages the owner → owner deletes → janitor logs path. A `ratatui` panel in the host shell renders the in-kernel `loop_state` of every agent in real time.

**Evaluation matrix we will publish in `EVAL.md`:**

| Metric | How measured | Target |
|---|---|---|
| `sys_tool_call` round-trip | `rdtime` ticks before/after, 10k iterations | <2× a no-op `getpid` |
| `query_file` vs path scan | wallclock, N=1000 files | ≥50× speedup |
| ACR read vs `sys_context_query` | `rdtime` ticks, 1k reads | ≥20× speedup |
| Idle agent CPU | QEMU `info cpus`, 60s wait | <5% on a single hart |
| Soak (10 agents, 1 hour, 100k tool_calls) | panic / leak / RSS drift | zero panics, RSS Δ <1MB |

Grading dimensions map cleanly: innovation (30%) covered by ACR + TLV-MCP bridge + path-predictive prefetch (a stretch goal in Sprint 8); completeness (20%) covered through Sprint 7; code quality (25%) by the `kani` audit + `clippy::pedantic` clean tree + small unsafe surface; doc (25%) by mkdocs site with the Mermaid set.

## 5. Risks & mitigations

- **Scope creep on the open-ended 25%.** This is the #1 killer for any open-design topic. Mitigation: lock the demo scenario at Sprint 1 (Scenario B), forbid new syscalls after Sprint 6, treat creative-extension items as Sprint 8 *opt-in*.
- **Unsafe-Rust bugs in the kernel/user shared ring.** Mitigation: confine all `unsafe` to one file (`kernel/src/agent/acr.rs`), enforce via `#![forbid(unsafe_code)]` elsewhere, run `miri` against a host-mocked version of the ring weekly.
- **rCore upstream drift.** Tutorial-v3 `main` last saw 460 commits and is reasonably quiet, but a sync mid-project costs days. Mitigation: pin a SHA at Sprint 1, only sync if we need an upstream bugfix.
- **Tool-call protocol bikeshedding.** Mitigation: ship the TLV in Sprint 3 *behind a version byte*; do not negotiate "is JSON better" past Sprint 4.
- **Multi-agent IPC race conditions.** Mitigation: a single coarse-grained `Mutex` over the event bus in v1 — premature optimization here loses code-quality points more than wins them.
- **Demo flakiness on judging day.** Mitigation: pre-record the canonical run, ship a `make demo-deterministic` target with `qemu -icount shift=1,sleep=off`.

## 6. Score & rationale

**Final score: 9/10.**

Why so high: the base kernel choice is settled, the toolchain is known-good, every base task maps to a concrete data structure we have already named, the advanced tasks are bounded, and the team can dogfood by pointing Claude Code itself at the resulting kernel and watching it use the new syscalls (rare, satisfying, also a strong demo beat). The 25/25 Stage 1 score reflects judge preference for "Agent-native infrastructure" themes — this is the moment to ride that wave. Compared to the median functional-track project, this one has unusually crisp acceptance criteria per task and an obvious narrative arc.

Why not 10: the topic is **engineering-flavored** (`tags: 工程型`) and there is real risk that two-three other teams independently converge on a very similar TLV-syscall design — defensible originality will hinge on the ACR layout and the path-predictive prefetch stretch goal, neither of which is a slam-dunk. The B-level grade ceiling also caps the absolute prize value vs. an A-level academic topic.

**Recommendation: green-light. Start Sprint 1 the week the team is assembled.**
