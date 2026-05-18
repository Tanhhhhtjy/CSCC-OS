# proj52 Feasibility Report — RVV-Accelerated "Intelligent" RISC-V OS

**Track:** 2026 CSCC OS Functional Challenge (Functional track, "智能" / 创新型)
**Mandated base:** RISC-V 64-bit, core code **C**, OpenSBI firmware, QEMU rv-virt / Gem5 / real board.
**Refs:** RISC-V Privileged spec, RVV 1.0 spec, 陈海波 *模型原生操作系统* (CCF 2025), 唐楚哲 *机器学习赋能系统软件* (计算机研究与发展, 2023), KML (USENIX ATC '22).
**Stage 1 score:** 16/25 — failed C=1 ("intelligent" is hand-wavy, demo-only risk).

---

## 1. Technical path

The Stage-1 critique is correct: "intelligent OS" without a measurable target is a poster project. The reframe is to pick **one kernel control loop** where (a) the input is a real, low-rate time series the kernel already observes, (b) the output is a single tunable knob, and (c) a published baseline exists to beat. Everything else is wallpaper.

**Base kernel.** Three live candidates: **xv6-riscv** (C, ~10k LoC, RV64, sv39, OpenSBI-compatible), **rCore-Tutorial-v3** (Rust, ~4k LoC — *disqualified* by the "core code must be C" rubric), or a from-scratch C kernel à la the eduxiji 2023-2024 cohort. We pick **xv6-riscv as upstream base with an out-of-tree `ml/` subsystem and an `agent_sched` replacement scheduler**, because: it is C, the boot path / trap handler / sv39 walk are upstream-clean and already well represented in any LLM's training corpus (load-bearing for the AI-assist verdict in §2), and it gives us 8-10 weeks of subsystem work rather than 16 weeks of "where does `_start` live". A fork-from-scratch wins zero points and loses two months.

**Where ML moves the needle — pick TWO, ship ONE+0.5.**

1. **Scheduler quantum + class selection (primary).** Replace xv6's round-robin with a 3-class scheduler (interactive / batch / I/O-bound) where class assignment is driven by a *static-weights* perceptron over per-process features `(avg_run_burst, voluntary_yield_rate, page-fault_rate, last_syscall_class)`. 4 features × 3 classes = 12 weights, all int8, inference is a 12-MAC dot product per scheduling decision. Trained **offline** on traces collected from xv6 user binaries (the included `usertests`, `forktest`, plus a synthetic mix), shipped as a static blob `model_sched.bin`. Output: class label, which selects a quantum (5/20/2 ticks) and a runqueue priority.
2. **Read-ahead window predictor (secondary).** Per-file sliding-window read-pattern classifier (sequential / strided / random / mixed) → window size ∈ {0, 4, 16, 64} blocks. 8 features, 4 classes, same shape, same blob format. Hooks the buffer cache `bread` path.

ML kernels — *if* time allows and the FPGA/board exposes vector — are vectorized via **RVV 1.0 intrinsics** (`__riscv_vsetvl_e8m1`, `__riscv_vle8_v_i8m1`, `__riscv_vmul_vv_i8m1`, `__riscv_vredsum_vs_i8m1_i32m1`). With a 12-dim model the scalar version is **already** under 2% CPU; RVV is for the (proposed but optional) per-page LSTM-light page-replacement stretch and for honoring the topic's "RVV-accelerated" framing in the writeup. Stretch only.

**Kernel-mode RVV is the load-bearing subtlety.** RVV state is huge (VLEN/8 × 32 registers; on a `vlen=128` core that's 512B; on SiFive P670 / SpacemiT K1 with VLEN=256, 1 KiB). xv6 currently does not save vector state on context switch. Three options, in order of preference:

- **(A) Lazy save with a kernel-only "RVV critical section"** — disable preemption around the ~50-cycle inference, restore `vstart/vtype/vl` on exit, never let user vector state collide. This is what Linux's `kernel_vector_begin()/end()` did when it merged into 6.8 (Andy Chiu, Greentime Hu, 2024); xv6 has no preempt-disable primitive but interrupts-off + a soft flag suffices for a uniprocessor demo. **We pick this.**
- (B) Save full RVV context on every trap. Doubles trap-frame size, hurts all syscalls — unacceptable for an "intelligent OS" demo whose headline metric is overhead.
- (C) Punt RVV to user space via a syscall — kills the "in-kernel ML library" thesis.

The trap frame in xv6 (`struct trapframe`, `kernel/proc.h`) gets a sibling `struct vec_frame` carrying `vstart, vxsat, vxrm, vcsr, vl, vtype` + the actual register file, allocated only for processes that touched RVV (lazy-VFP flag in `p->vec_used`). Saved on `kerneltrap`/`usertrap` only when `vec_used` AND target context will run user vector code. The kernel-side ML inference itself runs under `vec_begin()` and saves whatever the user had on entry.

**OpenSBI patching.** We add **one vendor SBI extension** at `EID = 0x09000052` (custom range): `sbi_intelligent_telemetry(read_perf_counters_subset)` to expose `mcycle/minstret/mhpmcounter3..` to S-mode without messing with `mcounteren`. Authored as a single `.c` under `lib/sbi/sbi_ecall_intel.c` registering an `sbi_ecall_extension` with `extid_start = extid_end = 0x09000052`. This is genuinely small (~120 LoC) and gives us a defensible "we touched firmware" beat. (Production-grade RV64 deployments expose these via `mhpmcounter` + S-mode `scounteren`, but routing through SBI keeps the patch contained and is what the rubric expects.)

**Toolchain.** `riscv64-unknown-elf-gcc 14.2` (RVV 1.0 intrinsics are spec-frozen-1.0 in GCC 14 per the GCC 14 changes page; LLVM 19 also OK), QEMU 9.0 with `-cpu rv64,v=true,vlen=128,elen=64,vext_spec=v1.0` (TCG-accurate but slow; spike for cross-check), OpenSBI v1.6+ (v1.8.1 is current as of Jan 2026). FPGA/board fallback: **SpacemiT K1** (BPI-F3) or **Kendryte K230** — both ship RVV 1.0 silicon, both bootable with OpenSBI. We will *not* assume board access; QEMU is the contract.

## 2. AI-capability assessment (Claude Code on RV64 kernel-C)

Concrete 1-5 ratings, based on what we've actually probed and what the public corpus shows:

| Subsystem | Rating | Notes |
|---|---:|---|
| RV64 boot + trap handler in C+asm | **4** | xv6-riscv is in every code LLM's training set; `_entry`, `start.c`, `trampoline.S`, sv39 walk are reproduced near-verbatim on demand. Hallucination risk is on **rare CSRs** (`henvcfg`, `senvcfg`, `mseccfg`) where the model fills plausible-looking names; mainline CSRs (`mstatus, sstatus, satp, scause, stvec, sepc, sip, sie, mhartid`) are reliable. `sext.w` semantics are usually correct; `addiw` vs `addi` errors do appear in generated boot code about 1 time in 10 — must be caught by `qemu -d in_asm,int` diff against a known-good xv6 boot. |
| OpenSBI vendor-extension authoring | **3.5** | `sbi_ecall_extension` registration boilerplate is well-known; the model occasionally mis-uses the older `sbi_ecall_register_extension` signature from OpenSBI <0.9. The `extid_start/extid_end` vendor slot convention is reliable. Code review by a human who has read `lib/sbi/sbi_ecall.c` once is sufficient. |
| RVV 1.0 intrinsics in **kernel** | **2.5** | This is the weakest cell. RVV 1.0 was frozen late 2021; training data is heavily polluted by **RVV 0.7.1** intrinsics (THead C906/C910 style: `vsetvli a0, t0, e32,m1` inline asm, `vfmacc.vv`, no `__riscv_` prefix). Claude/GPT will *frequently* produce a mix: 0.7.1 inline asm sprinkled into a `riscv_vector.h` intrinsic function. Kernel-mode vector-state save/restore is rarely seen in the corpus — the Linux `kernel_vector_begin/end` patch series (6.8, 2024) is recent and lightly indexed. Expect 50% of first-shot vector code to need rewriting. Mitigation: hand-write the 3 vector kernels (dot-product, ReLU, vector-max-pool) from the spec, use AI only for the scalar fallback. |
| In-kernel ML inference engine (int8 GEMV, fixed-point softmax) | **4** | This is small-matrix DSP code, well-represented; AI excels at it. Risk is on integer overflow guards (`int32_t` accumulator for `int8 × int8 × N` is fine for N≤2^15, generated code sometimes uses `int16_t` and silently overflows). |
| Evaluation harness, trace generators, `expect`/CI | **4.5** | Standard QEMU + serial-log + Python; reliable. |
| **Aggregate verdict** | | A 3-person team with AI assistance can ship items 1, 2, 4, 5 in 4 months. **Item 3 (RVV in kernel) is the single biggest schedule risk** and the right move is to descope it to a stretch goal, keep the *interface* RVV-shaped (`vec_begin/vec_end` wrappers, intrinsic-style C even if scalar) so the "RVV-accelerated" narrative survives even if only the scalar path lands. |

Comparable benchmarks of AI-assisted RV64 kernel work in the open: rCore-Tutorial-v3 (Rust) has multiple known-good "I used Cursor/Claude to add chapter X" lab solutions floating around — Rust + rCore is the easy mode. **xv6-riscv** has a much smaller pool of AI-assisted public forks; the few we found are tutorial-grade lab solutions, not subsystem additions. **starry-os** and other Rust ports are not relevant to a C track. **Net: AI assistance gives ~1.4× speedup on xv6-style C, ~2× on rCore-style Rust** — a non-trivial penalty for the C mandate but not prohibitive.

**Honest verdict:** 4 months with AI assistance closes the gap **for the descoped target** (scalar inference, RVV-shaped interface, scheduler + read-ahead). It does **not** close the gap for the maximal target (real RVV kernels, page-replacement LSTM, all three demoed subsystems). Plan for the descoped target; bank the rest as stretch.

## 3. Four-month sprint plan (8 sprints × 2 weeks)

**Sprint 1 (W1-2) — Baseline & instrumentation.** Fork xv6-riscv at upstream SHA, build green on QEMU 9.0 + OpenSBI 1.8.1, add `make trace` target that boots, runs `usertests`, dumps serial + `mcycle/minstret` to a `boot.trace`. Mermaid architecture diagram. Acceptance: `make ci` < 60s, deterministic with `qemu -icount shift=auto`.

**Sprint 2 (W3-4) — Telemetry & feature pipeline.** Add per-proc counters (run-burst histogram, yield count, page-fault count, last-syscall class) to `struct proc`. Expose via a new `sys_proc_stats` syscall + dump to `/dev/trace` virtio-console. Vendor SBI extension `0x09000052` for hart-level perf counters. Acceptance: 1-hour `usertests` trace produces a CSV the offline trainer eats.

**Sprint 3 (W5-6) — Offline training harness.** Python sidecar: read CSV, label scheduling classes by post-hoc analysis (run-burst < 1 tick → interactive, etc.), fit a 12-weight int8 perceptron, emit `model_sched.bin` (header + weights + bias + scale). Unit-test the C inference against the Python reference on 1000 synthetic vectors, max abs delta = 0 (deterministic fixed-point). Hard gate: bit-exact parity.

**Sprint 4 (W7-8) — In-kernel inference engine + scheduler hook.** `kernel/ml/`: blob loader (linked as `.rodata` via `objcopy --rename-section`), scalar GEMV, argmax. Replace round-robin with class-driven quantum. Acceptance: `forktest`, `usertests`, `stressfs` all green; ML inference ≤ 1.5% of total cycles on a mixed-workload trace (measured via `mcycle` deltas around `mlsched_classify()`).

**Sprint 5 (W9-10) — Workload generator + measurable-win run.** Build a "mixed trace" benchmark: 4 interactive + 4 batch + 4 I/O processes spawned by a master `bench` program. Compare round-robin baseline vs ML-driven on (i) interactive p99 response time, (ii) batch throughput, (iii) I/O completion. Target: **≥10% improvement on a composite score** the eval harness pre-registers in `EVAL.md`. If not hit, debug; do not move on.

**Sprint 6 (W11-12) — Read-ahead predictor (subsystem #2).** Same shape, smaller scope, hooks `bread`. Bench against fixed read-ahead = {0, 16, 64} on a synthetic sequential+random mix. Target: ≥15% IOPS or ≥10% wall-clock on a mixed pattern. This sprint can slip 1 week into Sprint 7 without killing the demo.

**Sprint 7 (W13-14) — RVV path (stretch) + kernel-mode `vec_begin/end`.** Add `vec_frame`, lazy-save on context switch, three RVV intrinsic kernels (dot, ReLU, argmax) behind a config flag. Cross-check against scalar via bit-exact comparison on the same input vector. If kernel-mode RVV proves too hairy in QEMU TCG (frequent: vstart/vl interaction with traps), ship the *interface* with a scalar implementation behind it and document the gap honestly. **Honesty is worth more than a half-broken RVV path.**

**Sprint 8 (W15-16) — Hardening, paper, demo.** Run a 6-hour soak, ASan-equivalent via QEMU `-trace` and a leak-detection pass, write the 12-page design doc (Mermaid: arch / trap path / vec_frame layout / inference dataflow), 7-min screencast, freeze tag `v1.0-cscc`.

Cuts in priority order if behind: (1) drop RVV path entirely, (2) drop read-ahead predictor, (3) drop vendor SBI extension and use direct `mcycle` reads from S-mode (requires `scounteren` set, fine). The scheduler ML demo is the load-bearing deliverable; everything else is decoration.

## 4. Skills to author

- **`rv64-kernel-scaffold`** — codifies the xv6-riscv-on-OpenSBI-1.8 boot recipe: `_entry` → `start.c` → `main.c` skeleton, sv39 walk, trampoline, `usertrap/kerneltrap`, a known-good `Makefile` with QEMU 9 + `-cpu rv64,v=true,vlen=128`, plus a "lint" that catches the common AI failure modes (mixed RVV 0.7.1/1.0, missing `addiw` vs `addi`, wrong `satp.MODE`). Saves a week on Sprint 1.
- **`opensbi-vendor-extension-template`** — `lib/sbi/sbi_ecall_*.c` boilerplate for the 0x09xxxxxx vendor range, with EID/FID conventions, S-mode caller stub, registration into the platform's extension table. Reused for any future kernel↔firmware ABI work in the lab.
- **`rvv-kernel-context-handler`** — the `vec_begin/vec_end` discipline: trapframe layout for `vec_frame`, lazy-save flag, interrupt rules ("no vector in IRQ context"), VLEN-agnostic save loop. References the Linux 6.8 patch series for the canonical shape. This is the highest-leverage skill — any future kernel-mode-vector work in C (RISC-V or ARM SVE) reuses 80% of the discipline.

## 5. Demo + evaluation — what makes it "intelligent" vs "configured"

The poster-project trap is exactly: "we set quantum = 20 ms and called it intelligent". The escape is a **pre-registered comparative protocol**:

**Demo.** Live in QEMU: boot xv6, run `bench mixed`, record metrics. Reboot with `make CONFIG_ML=0`, run same `bench mixed`, show round-robin baseline. Diff visualized in a `ratatui`-style serial-console panel: latency p50/p99 + throughput, live, side-by-side. Then `make CONFIG_ML=1 CONFIG_ML_MODEL=adversarial.bin` — load a deliberately bad model — show the system gets *worse*, proving the kernel is actually consulting the model, not us hand-tuning a knob.

**Measurable win, pre-registered in `EVAL.md` before Sprint 5:**

| Metric | Baseline (RR) | Target (ML) | Method |
|---|---:|---:|---|
| Interactive p99 latency on mixed workload | X ms | ≤0.85·X ms | 1000 syscall round-trips under load |
| Batch throughput (ops/s) | Y | ≥1.10·Y | `forktest` + `stressfs` mix |
| ML inference overhead | n/a | ≤2% CPU | `mcycle` delta around `mlsched_classify()` |
| Model storage | n/a | ≤4 KiB | size of `model_sched.bin` |
| Adversarial-model regression | n/a | ≥20% worse | proves the loop is closed |

The **adversarial-model row** is the single most important grading-day beat. It directly refutes the Stage-1 C=1 critique: it is not possible to fake "the kernel is using the model" when swapping the model demonstrably breaks the system. This converts "intelligent" from rubric-vague to falsifiable.

## 6. Top 3 risks + mitigations

1. **"Demo-only" / "configured-not-intelligent" critique (Stage 1's C=1 risk).** Mitigation as above: the adversarial-model swap is built into Sprint 5's deliverable, not bolted on at the end. Plus: the model is **trained offline** on traces, not hand-set — the trainer is a tracked artifact, the training loss is in `EVAL.md`, the trained weights are reproducible from `make model`. Reviewers can re-train.
2. **Kernel-mode RVV trap-handling subtleties (vstart re-entry, VLEN drift, IRQ rules).** Mitigation: descope RVV to a stretch goal with a scalar fallback behind the same `vec_begin/end` interface (§3 Sprint 7). Lift the discipline from Linux 6.8's series rather than re-deriving it. If RVV lands, cross-check scalar-vs-RVV outputs bit-exact on the same inputs in CI — this catches all the vstart/vl bugs deterministically.
3. **Offline-trained model ships as a static blob, which is "not actually online learning".** This is a soft risk — judges may ask "where is the self-adaptation?". Mitigation: ship **two** modes — (i) static blob (the reliable demo), (ii) an **online weight-update** path that nudges weights via a 1-step gradient when post-hoc reward (measured 1s later) disagrees with prediction. The online path lives behind a flag, runs only on the scheduler perceptron (12 weights → trivially safe to mutate), and produces a learning-curve plot over a 1-hour run. This gives a defensible "self-tuning" answer without staking the demo on it.

Honorable-mention risks: QEMU RVV TCG inaccuracy vs Spike (mitigate by Spike cross-check in CI); xv6 single-hart limit (the demo is single-hart by design, advertised honestly); IP-licence concerns of using vendor counters (we don't — SBI ext is in vendor range).

## 7. Final pick score: **6.0 / 10**

**Versus shortlisted alternatives:**

- **proj61 (Agent-OS Kernel Extension, 9.0/10):** clearer subsystem boundaries, Rust upstream, B-level engineering grade with crisper rubric. Strictly easier to ship and to grade well. proj52 loses on both counts.
- **proj18 (Small-OS Analysis Agent, 7.5/10):** A-level academic, no kernel coding, totally different risk profile (citation correctness vs kernel correctness). Different bet, not directly comparable on capability axis.
- **proj41 / proj53:** if those reports converge near 6-7, proj52 is in the same band — interesting topic with a real "demo-only" tail risk.

**100-word rationale.** proj52's *headline* is exciting and timely (RVV + in-kernel ML + RISC-V is exactly where the field is heading per Chen Haibo's *model-native OS* line of work), but the **C=1 critique is structural, not cosmetic**: "intelligent OS" without a falsifiable closed loop is poster-grade. We can fix it — adversarial-model swap + pre-registered metrics + offline-trained-with-online-fine-tune — but doing so means descoping RVV from headline to stretch, which dilutes exactly the differentiator the topic puts in its title. The result is a **defensible 6/10 project**: real OS work, real ML loop, honest scope. It is **not** a 9 like proj61, because the demo carries irreducible "is this intelligent or just configured?" ambiguity that proj61 doesn't have. **Recommendation: do not green-light over proj61; consider as a second pick only if a teammate is uniquely keen on kernel-mode vector work.**
