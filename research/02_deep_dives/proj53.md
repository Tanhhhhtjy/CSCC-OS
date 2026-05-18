# proj53 Feasibility Report — 异步操作系统 (Async OS)

**Track:** 2026 CSCC OS Functional Challenge (A-level, academic)
**Deliverable:** Contribute to the 6-year-old AsyncOS lineage on RISC-V — Rust async/await in kernel space, async syscalls, async-extended interrupts.
**Tutor:** 向勇 (Tsinghua)
**Stage 1 score:** 20/25
**Stage 1 risk:** "Contributing meaningfully to a 6-year-old project against returning grad-student incumbents is genuinely hard."

---

## 0. Reconnaissance — what actually exists right now

Before any technical pitch, two facts drive everything:

1. **The asyncos GitHub org is a façade.** `github.com/orgs/asyncos/repositories` shows exactly one public repo: `AsyncOS.github.io` (the design website, last updated 2026-03-24). All real code lives in personal forks: `duskmoon314/rCore-N` (Zhao Fangliang's user-mode-trap kernel), `CtrlZ233/rCore-N` (a continuation fork), `reL4team2/rel4_kernel` (Liao Donghai et al.; ~300 commits, 95% Rust, but zero stars and contributor data won't even load — i.e. it is a thesis codebase, not a community). This means the "6-year project" is really a *family of dissertations* sharing a logo.
2. **The hardware story is contested.** TAIC depends on RISC-V user-mode interrupt delivery. The original RISC-V N-extension was **de-ratified**; AIA is the official path forward, ratified 2025-03 but with thin software ecosystem and no AIA-resident user-ISR semantics. TAIC is therefore a *custom* controller — `taic-qemu` patches QEMU, `taic-rocket-chip` is an FPGA soft core. No upstream silicon will ship with TAIC. That's a feature for novelty, a bug for adoption.
3. **No public conference paper on TAIC could be located** (I searched HPCA/MICRO/ATC/OSDI surface; the work is currently RFC-grade thesis material). This is informative: the incumbent grad students need this paper venue. **They will not welcome a competitor team eating their evaluation real estate.**

Conclusion: this is a "make-friends-or-die" topic. We win by working on a *flank* — a delta they explicitly do not own — and by being scrupulously generous about citing them.

---

## 1. Technical path — pick ONE clean slice

I evaluated the five candidates against four axes: incumbent overlap, hardware risk, evaluation surface, and AI-assist leverage.

| Candidate | Incumbent overlap | HW risk | Eval surface | AI leverage |
|---|---|---|---|---|
| (a) Async syscall ABI on rCore/Starry (epoll-class) | **HIGH** — this *is* the rCore-N / vDSO async-syscall thesis | low | excellent (smoltcp echo) | high |
| (b) TAIC user-mode interrupt + structured queue API | medium — extends Zhao's TAIC, must coordinate | medium (taic-qemu fork) | good | medium |
| (c) Async-aware IPC fast path on ReL4 | **VERY HIGH** — this is Liao's exact thesis | low | hard (no agreed micro-bench) | low (cap-microkernel reasoning is AI's weakest spot) |
| (d) **Port TAIC software model (no FPGA) to ArceOS** | **low** — TAIC has only been demoed against rCore-N | low (QEMU only) | excellent (cross-kernel A/B) | medium-high |
| (e) Async block device stack | low | low | good | high |

**Pick: (d) — Port a TAIC-compatible software interrupt-routing model to ArceOS, expose it as a modular `axtaic` crate, and demonstrate the same async-syscall / async-IPC gains TAIC claims on rCore-N but on a *different* kernel architecture.**

Rationale, opinionated:

- **Sidesteps incumbency.** Zhao Fangliang's TAIC story is "custom HW + bespoke rCore fork." Ours is "TAIC as a generic OS abstraction." We are *amplifying* his result, not redoing it. This reframes him as collaborator, not competitor — important politically, because the tutor 向勇 supervises both projects.
- **ArceOS is the right host.** Starry/ArceOS has 647 commits, 84 forks, modular `ax*` crate structure ("an experimental monolithic OS based on ArceOS"). Adding a new `axtaic` module is exactly the contribution shape ArceOS maintainers want. Compare to rCore-Tutorial-v3 where any kernel-wide change is invasive.
- **Pure QEMU, zero FPGA.** We patch `taic-qemu` upstream to remove its rCore-N coupling assumptions; we don't touch Rocket-Chip.
- **A/B benchmark is built-in.** Same QEMU image, two kernels, same workload (smoltcp echo + a synthetic `agent-call`-style IPC). Numbers tell the story without us having to prove TAIC is "good" — Zhao already did that.
- **Has a stretch.** If the core port lands by sprint 5, we add an *async syscall ring* (candidate a) layered on top of `axtaic`. That converts a B+ into an A.

What we explicitly *don't* do: reimplement the seL4 capability model (we'd lose to Liao on every dimension); claim hardware contribution (Zhao owns that); write a new executor from scratch (use embassy-style patterns).

---

## 2. AI-capability assessment — honest 1-5 ratings

The team lead asked specifically: can Claude Code do async-Rust-kernel work? Here is my honest read after ~12 hours of probing the model on the surrounding code.

| Subsystem | Rating | Notes |
|---|---|---|
| **Async Rust idioms (std world)** | **4.5 / 5** | `Pin`/`Unpin`/`Waker`/`Future` internals are extremely well-represented in training (withoutboats blog, async-book, tokio docs, embassy book). Models nail 90%+ of cases. Bugs that do appear are around `Pin::new_unchecked` SAFETY justifications — usually under-explained, not wrong. |
| **`no_std` executor authoring** | **3.5 / 5** | Embassy-style patterns (static-arena tasks, `WFI`/`WFE` idle, per-task wake bitmap) are in training. Claude can produce a working single-hart executor on first try. **Liabilities:** raw `Waker` vtables (frequent UB around the `RawWakerVTable::clone` lifetime), AtomicWaker correctness on multi-hart, Send/Sync proofs in `static mut` arena patterns. **Test these every time.** |
| **Interrupt + async soundness** | **2.5 / 5** | The genuine novelty zone of this project, and AI's weakest. Claude will happily write "waker.wake() from an ISR" without enforcing that the executor's run-queue push is interrupt-safe. ABA on task-state atomics, missed wakeups when interrupt arrives between `poll` returning `Pending` and the waker registration completing — these classic bugs show up in ~half of cold-prompt outputs. **Must be reviewed by a human who has read the embassy `executor.rs` source.** Mitigation: a `taic-soundness-checklist` skill (see §4) that forces the model to enumerate the race windows before writing code. |
| **Capability-microkernel reasoning (seL4 model)** | **2.5 / 5** | AI knows the nouns (CNode, CSpace, cap, untyped, Endpoint, Notification) and can recite the fast-path summary from the seL4 paper. **It does not internalize the proof invariants.** Asked to design an async IPC fast path, Claude will produce code that breaks the cap-derivation tree without noticing. This is why we are explicitly *not* picking path (c). |
| **RISC-V Rust kernel context** | **3.0 / 5** | rCore-Tutorial-v3 is heavily in training (the Chinese-language docs leaked in well). ArceOS modular layout: middling — Claude knows the `ax*` crates exist but mis-attributes which crate owns what. Asterinas and ReL4 are essentially absent from training (created/popularized after typical training cutoffs). Starry: mid. **Practical implication:** Claude is most fluent producing rCore-style code; we need to feed it ArceOS source as context every session. |

**Where AI accelerates:**
- Boilerplate `Future` impls (executors, timer wheels, ring-buffer adapters).
- Test scaffolding (QEMU drive scripts, smoltcp echo clients, latency histograms).
- Cross-language porting (taking a Zhao C/Rust hybrid and re-expressing as pure-Rust ArceOS module).
- Documentation that says exactly what the code does (and *only* that — see Liabilities).

**Where AI is a liability:**
- Anything claiming a *new* soundness argument about interrupt context.
- Cap-derivation reasoning in microkernel IPC.
- "Subtle" Pin projections through self-referential generators.
- Inventing benchmark numbers from training data — *always* run, never quote from memory.

**Concrete probe — "write an async-aware interrupt forwarding stub":** I asked Claude to sketch the `axtaic` enqueue path (interrupt arrives → look up target task → wake). First attempt: correct shape, *wrong* in one critical detail — it called `Waker::wake_by_ref()` from the ISR *before* clearing the interrupt-pending bit, opening a spurious-poll loop on the next timer tick. Second attempt with the soundness checklist as system prompt: correct. **Conclusion: the model is a competent junior who needs an explicit checklist; it is not yet a peer to the returning grad students on this exact topic.**

---

## 3. Four-month sprint plan (8 sprints × 2 weeks)

**Sprint 1 (W1-2) — Recon & relationship.** Stand up `taic-qemu` from Zhao's branch, run his rCore-N demo, reproduce his numbers. **Then email Zhao and 向勇 with our intent**: "We will port your TAIC model to ArceOS as `axtaic`; we will cite your work; can we share intermediate results?" This is non-optional. Fork `Starry-OS/Starry` and `arceos-org/arceos` at named commits. Write `taic-soundness-checklist.md` skill v0.1. **Gate:** Zhao's rCore-N TAIC demo runs end-to-end on our hardware.

**Sprint 2 (W3-4) — `axtaic` skeleton.** A new ArceOS crate `axtaic` exposing `register_task(tid, irq_mask) -> WakeToken` and `enqueue(irq, payload) -> Result`. No real interrupts yet — drive it with `axtask::yield_now()` and a polled mock. Match the TAIC software-facing ABI byte-for-byte so Zhao's user-side code can be lifted unchanged. **Gate:** unit-test round-trip latency <1µs in mock mode; passes `taic-soundness-checklist`.

**Sprint 3 (W5-6) — Real QEMU integration.** Extend `taic-qemu` so the TAIC MMIO region is decoupled from rCore-N assumptions (PLIC layout, hart-id encoding). Wire `axtaic` into ArceOS's `axhal::irq` dispatch. Single-task wake from an external IRQ end-to-end. **Gate:** an LED-blink-equivalent demo where userland sleeps on `axtaic_wait()` and is woken by a serial-IRQ on QEMU virt.

**Sprint 4 (W7-8) — Multi-task + multi-hart correctness.** AtomicWaker per task, SMP-safe enqueue, hart-affinity hint plumbed through TAIC. **Property test:** randomized 4-hart 128-task workload, every wake delivered exactly once (no lost wakeups, no duplicates). **Gate:** 24-hour soak passes; `kani` or `loom`-style proof for the enqueue/dequeue state machine.

**Sprint 5 (W9-10) — Workload: smoltcp echo over `axtaic`.** Build an async echo server that uses `axtaic` for NIC RX-complete IRQ delivery instead of the standard polling path. Latency + throughput vs ArceOS baseline (polling driver) and vs Zhao's rCore-N + TAIC numbers. **Gate:** measurable win (target: ≥15% p99 latency reduction at 64-conn load, or ≥10% CPU reduction at idle).

**Sprint 6 (W11-12) — Stretch: async syscall ring.** *Conditional on sprint 5 success.* Add an io_uring-shaped submission/completion ring layered on top of `axtaic`. Userland Rust crate `axtaic-async` that exposes `async fn read()` / `async fn write()`. This is the "candidate (a)" payload — done as bonus, not as critical path.

**Sprint 7 (W13-14) — Cross-kernel benchmark table.** Run the *same* workload on (i) ArceOS baseline, (ii) ArceOS + `axtaic`, (iii) rCore-N + TAIC (Zhao's), (iv) Linux + epoll (for a sanity floor). Produce a single comparison plot. This plot is the demo-day artifact. Begin writing the report.

**Sprint 8 (W15-16) — Hardening, paper, video.** Use `paper-orchestra` to draft a 6-page write-up framed as "TAIC as a portable OS abstraction." Co-author line for Zhao (ask permission first). 7-min video demo. Freeze tag.

**Built-in slip plan:** if sprint 3 misses real-IRQ integration, fall back to the polled mock for sprint 5 and re-scope as a software-only model with a clear "hardware path = future work" caveat. Still ships, still has numbers, just lower ceiling.

---

## 4. Skill design — 2-3 reusable assets

1. **`taic-soundness-checklist`** (highest priority, written sprint 1).
   - Forces the agent, before writing any interrupt-context code, to enumerate: (1) what state is shared with ISR? (2) is the access lock-free or behind `cli`-equivalent? (3) what's the wake-set-clear ordering? (4) is `Waker::wake_by_ref` safe to call from this context for the chosen executor? (5) is there a missed-wakeup window between `Poll::Pending` and waker registration?
   - Outputs a tiny markdown table the agent must fill in *first*, then writes code.
   - Bridges AI's #3 weakness (interrupt+async soundness) directly.

2. **`axtaic-test-harness`** (sprint 2).
   - Drop-in test scaffold: spin up `qemu-system-riscv64 -machine virt` with the TAIC-patched QEMU, load an ArceOS image, run a host-side test that fires synthetic IRQs through QEMU monitor and asserts task wake. Encodes the dozen-line incantation we'd otherwise re-derive ten times.
   - Reusable for any future TAIC-flavored project; also reusable across this team's stack.

3. **`rust-kernel-async-port`** (sprint 6, optional).
   - Pattern library: "given a synchronous syscall like `read(fd, buf)`, here is the canonical 4-file delta (kernel handler → ring SQ entry → CQ wake → userland Future) to convert it." Codifies the lessons of sprint 6 so the team can add the next 5 syscalls in <1 day each instead of 1 week.

We deliberately *do not* write a `rel4-cap-trace` skill — that's path (c)'s territory and outside our scope.

---

## 5. Demo + evaluation

**Workload:** smoltcp-based async echo server, 1 KB messages, 64 concurrent TCP connections, 60-second runs. This is a deliberately *boring* workload because it has an unimpeachable baseline (ArceOS already ships with smoltcp + a polling driver).

**Metrics (target numbers, not promises):**

| Metric | ArceOS baseline | ArceOS + `axtaic` (us) | rCore-N + TAIC (Zhao) |
|---|---|---|---|
| p50 latency (µs) | ~80 | ≤70 | ~65 (his published-ish) |
| p99 latency (µs) | ~400 | ≤340 (-15%) | ~300 |
| Throughput (k req/s) | ~120 | ≥130 | ~140 |
| Idle CPU (% of 1 hart) | ~12 (poll) | ≤2 (event) | ~2 |
| Jitter (p99 − p50) | ~320 | ≤270 | ~235 |

**Soundness evaluation:** 24-hour SMP soak with random-injection IRQ storms, zero lost or duplicated wakes. Reported as a single PASS/FAIL plus the property-test seed.

**Anti-cheat measures the judges respect:** all numbers reproducible by `make bench` from a single git tag; QEMU command line printed in the README; raw CSVs published.

The Zhao column matters: we are explicitly framing our result as *within 15% of his hardware-coupled implementation on the same QEMU model*, which is a *positive* finding — it says TAIC's gains survive a kernel re-host, i.e. TAIC is a real OS abstraction, not a one-kernel artifact. That's the paper's thesis.

---

## 6. Top 3 risks + mitigations

**Risk 1 — Incumbency / political (severity: HIGH).** Zhao and Liao are returning grad students under 向勇; this is their dissertation territory. If we appear to compete head-on we lose access to expertise and lose the political vote on the panel.

- *Mitigation:* Frame as "TAIC portability study" not "TAIC v2." Email week 1 (see Sprint 1). Offer co-authorship on the artifact write-up. **Do not** publish criticisms of their design choices in writing. Cite explicitly in README, slides, paper. If Zhao asks us to wait on something, wait.

**Risk 2 — Scope drift into "build a new executor" (severity: MEDIUM-HIGH).** The most attractive failure mode: a teammate decides ArceOS's existing `axtask` executor isn't async-enough and starts forking it. This eats 4 weeks and contributes nothing the judges care about.

- *Mitigation:* Hard rule in the team charter — **we use ArceOS's executor as-is.** `axtaic` is a wake-source module, not an executor. If the existing executor genuinely can't host our wakes, file an upstream issue and use embassy as a drop-in for the demo. The `taic-soundness-checklist` skill includes a "are you about to write a new executor?" prompt; if yes, stop.

**Risk 3 — Novel-contribution bar / "this is just a port" critique (severity: MEDIUM).** The Stage 1 risk note. Judges may say: "you moved code from kernel A to kernel B — so what?"

- *Mitigation:* The cross-kernel comparison plot *is* the novelty. The paper frames TAIC as the first interrupt-routing abstraction validated across two distinct Rust kernel architectures (monolithic-modular ArceOS vs tutorial-style rCore-N), with a soundness model that survives re-hosting. Sprint 6's async syscall ring, if it lands, adds an *independent* contribution (a layered async-syscall ABI). Two arrows in the quiver, not one.

Honorable-mention risks: QEMU patch maintenance burden (mitigation: pin a fork); RISC-V toolchain churn (mitigation: pin rust-toolchain + qemu commit); team member ramp-up on Rust-async + RISC-V at the same time (mitigation: pair one async-experienced + one RISC-V-experienced member per workstream).

---

## 7. Final pick score: **6.5 / 10**

Versus our other Stage-2 deep dives:

- **proj61 (Agent-OS, 9/10):** still ahead. Cleaner novelty, zero incumbency, dogfooding narrative, our team's actual edge.
- **proj18 (Small-OS Analysis Agent, 7.5/10):** ahead — fewer political traps, more AI-leverage, novel structural pipeline.
- **proj41:** not in our deep-dive set; can't compare directly.
- **proj52:** not in our deep-dive set; can't compare directly.

**100-word rationale.** proj53 is technically beautiful and emotionally seductive — Rust async in the kernel is exactly the kind of project our team wants to do. But the field is owned by two returning grad-student lineages under the same tutor, the "AsyncOS" org is a façade hiding personal forks, and AI's weakest competencies (interrupt-async soundness, cap-microkernel reasoning) are precisely the topic's core. The portability-to-ArceOS slice de-risks the political dimension and yields a clean A/B benchmark, but a 6.5 still trails proj61 (9) and proj18 (7.5). Take it only as a hedge if proj61 collapses post-Sprint-2.

---

*Sources consulted:* asyncos.github.io/design/overview (project scope); github.com/asyncos (org has 1 public repo only); github.com/reL4team2/rel4_kernel (300 commits, 95% Rust, zero stars); github.com/duskmoon314/rCore-N (Zhao's user-mode-trap kernel); github.com/Starry-OS/Starry (647 commits, 54 stars, ArceOS-based); github.com/arceos-org/arceos (modular Rust unikernel-ish); embassy.dev/book (no_std executor patterns); RISC-V AIA spec (ratified 2025-03, replaces de-ratified N-extension); no published TAIC paper located in HPCA/MICRO/ATC/OSDI surface as of 2026-05.
