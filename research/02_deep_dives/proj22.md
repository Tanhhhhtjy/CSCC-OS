# proj22 Deep Dive — OpenHarmony DSoftBus LLM-driven Fuzzer

**Stage 1 score:** 22/25 · **Level:** B · **Tag:** 工程型 · **Tutor:** 康梓峰 (BUPT)
**Title (EN):** An Intelligent Fuzzing System for the OpenHarmony Distributed Soft Bus Protocol

## 1. Target restated

DSoftBus (`communication_dsoftbus`) is OpenHarmony's "super-terminal" connection plane: it handles **device discovery** (CoAP over multicast, BLE adv, USB), **authentication & bus building** (Huawei-Key Exchange-style HiChain handshake, AuthSession), **session/transmission** (TCP, UDP, BR, BLE GATT, ML-Lane scheduler), and **net builder / LNN** (Lite-Network gossip). It is C/C++, ~400 K LoC, runs as `softbus_server` daemon plus a per-app SDK that talks to it through IPC. The attack surface is multi-layer state-machine code with strict ordering (discovery → auth → channel-open → data), and tutoring/refdata explicitly point at it.

Organizer-provided resources (from `feature`):

- Cloud container with ≥3 simulated OpenHarmony nodes pre-networked, DSoftBus pre-built.
- Standardized debug interfaces for edge-coverage collection, crash monitoring, state tracing.
- Seed corpus that obeys protocol grammar, plus the API/state-graph doc.
- Baseline toolchain: AFL++ (incl. QEMU user mode), CodeLlama-7B with sample invocation code, coverage feedback pipeline.

Quantitative targets:

| Metric | Bar |
|---|---|
| Branch-coverage delta vs AFL++ baseline | ≥ +15 % |
| Reproducible bugs w/ POC | ≥ 2 |
| State/logic-related bugs (auth-bypass, state-confusion DoS, ...) | ≥ 1 |
| Engineering closure | seed → exec → triage automated |

## 2. State of the art (May 2026 snapshot)

| System | Venue | Core idea | What it is *not* |
|---|---|---|---|
| **Fuzz4All** | ICSE'24 | Autoprompting + LLM-mutation across 9 SUTs (GCC, Clang, Z3, CVC5, Qiskit, Java, Go ...). 98 bugs total. | Stateless. Generates *self-contained* programs, no protocol session. |
| **TitanFuzz** | ISSTA'23 | LLM generates DL-API test programs for PyTorch/TF. | Single-call API surface, no multi-step state. |
| **ChatAFL** | USENIX Sec'24 | Three LLM hooks on AFLnet: (i) grammar extraction, (ii) seed enrichment, (iii) plateau-breaking "novel state" prompt. Subjects: pure-ftpd, ProFTPd, Kamailio (SIP), Live555 (RTSP), Exim. | Generic, single-host; OpenAI GPT-3.5; no kernel-coverage; no distributed-multi-node corpus mgmt. |
| **CKGFuzzer** (ICSE'25, in `refdata`) | LLM + code knowledge graph → drives **harness** generation, not in-loop mutation. | Off-line harness synth, not stateful protocol. |
| **PromeFuzz** (CCS'25, in `refdata`) | Knowledge-driven harness generation. | Same — harness generation. |
| **LLM4Fuzz survey** (Huang et al. 2024) | Taxonomy: seed-gen / mutator / grammar / harness. Notes "protocol- and state-aware fuzzing remains an open problem". | — |

**The closest prior art is ChatAFL.** A serious OS-track entry must therefore be very clear about what it adds *beyond* ChatAFL — otherwise the contribution collapses to "we re-ran ChatAFL on a new target", which is a security-paper artifact at best and not an OS-track win.

## 3. Differentiation strategy vs ChatAFL

Five concrete deltas, each individually defensible:

### 3.1 First-class state-machine modelling for DSoftBus

ChatAFL re-derives "states" from server response codes (RFC-style status numbers) and treats the protocol as a black box. DSoftBus has **no such response-code regularity** — discovery is multicast CoAP, auth is binary TLV inside HiChain frames, session-open is IPC + binary headers. The OS-track entry should:

- Build an explicit **DSoftBus state graph** (~30 states across `disc → auth_session → channel_open → data → close`) from the organizer-provided state doc + static call-graph analysis of the `dsoftbus/core/` and `core/authentication/` source.
- Annotate each state with: required prior messages, fields the LLM is allowed to mutate, and the kcov/edge-coverage IDs that gate forward progress.
- Use the LLM not just to "break plateaus" but to **synthesize precondition messages** for any target state: prompt = "to reach state S = `AUTH_SESSION_VERIFIED`, the sequence so far is M1...Mk; produce a valid Mk+1 that advances the session".

This is the **State Machine as Prompt** idea — ChatAFL has no such structural prior because their targets are RFC-defined and known.

### 3.2 Kernel-coverage feedback via kcov (not just edge-bitmap)

DSoftBus crosses user/kernel boundaries (IPC, sockets, BR/BLE drivers). Pure edge-bitmap from afl-clang-fast covers only the user-space daemon. The differentiator is to **fuse three coverage sources** the organizer's pipeline already exposes:

1. AFL++ shared-memory edge bitmap (user-space, per-process).
2. **kcov** trace per `softbus_server` thread — captures kernel paths reached by IPC / socket syscalls. (kcov is in the OpenHarmony kernel since 5.0; the organizer's container almost certainly exposes `/sys/kernel/debug/kcov`.)
3. State-tag stream from the protocol's own event log.

The LLM's reward is the **joint novelty** across all three. This is a system-software contribution that a security venue would never bother to make — they care about CVEs, not about plumbing.

### 3.3 Distributed multi-node corpus minimization

The container hosts ≥3 nodes; a single test case is a **scripted multi-party exchange** (node A discovers, B authenticates, C joins as third lane). ChatAFL has no concept of this.

Plan:

- Represent a "test case" as a DAG of per-node message scripts plus a synchronization spec (barriers).
- Run AFL++ instances per node; share queue via a custom **distributed minimizer** that drops a script if removing it doesn't shrink the joint-coverage union (cmin-merge across nodes).
- LLM is prompted with the *multi-node* trace and asked to introduce one cross-node race ("now insert a duplicate channel-open from C while B is mid-handshake").

### 3.4 LLM-on-the-edge cost discipline

ChatAFL uses GPT-3.5 over the OpenAI API (slow, rate-limited, $ cost). The organizer ships **CodeLlama-7B locally**. The entry must measure and report:

- Mutations per LLM call (batch decoding, vLLM/exllama).
- LLM-share of total fuzz-cycle wall time — should be ≤ 20 %, otherwise traditional havoc wins by raw throughput.
- An adaptive scheduler: pure havoc when coverage is rising; LLM-burst only when the edge-bitmap saturates for N cycles. (This is the "plateau detector" from ChatAFL, but with **local-model cost accounting** that ChatAFL doesn't have.)

### 3.5 OpenHarmony-specific bug-class taxonomy

Score-wise, 30 % of the rubric is "depth and quality of bugs found", with an explicit ask for **state-related logic bugs** (auth bypass, state confusion → permission escalation). The entry should pre-commit, in the design doc, to four bug classes and instrument the harness to detect each:

| Class | Detector |
|---|---|
| Authentication bypass | Sanity-check post-condition: any `OnChannelOpened` callback fired before `AuthSession::VERIFIED` flag is set → flag. |
| Heap/UAF in TLV parser | AddressSanitizer + the existing OpenHarmony fuzz harnesses under `dsoftbus/.../test/fuzztest/` (provided in tree). |
| Resource-exhaustion DoS via LNN gossip | rate-limited monitor on `softbus_server` RSS / FD count. |
| Cross-session state confusion (channel from session A consumed by session B) | invariant on `(sessionId, channelId)` pair monotonicity. |

This pre-committed taxonomy maps 1:1 to the rubric's "中高危逻辑漏洞" requirement.

## 4. Architecture sketch (4 components)

```
+-----------------+      +-----------------+      +------------------+
|  Static prior   |      |  LLM Mutator    |      |  Coverage fuser  |
|  (state graph,  |----->|  (CodeLlama-7B  |<-----|  AFL bitmap +    |
|   call graph)   |      |   + vLLM)       |      |  kcov + state    |
+-----------------+      +--------+--------+      +------------------+
                                  |                        ^
                                  v                        |
                         +-----------------+               |
                         |  AFL++ custom   |---------------+
                         |  mutator + fuzz |
                         |  orchestrator   |
                         +-----------------+
                                  |
                              (3 nodes)
                                  v
                         +-----------------+
                         |  DSoftBus       |
                         |  multi-node     |
                         |  test bench     |
                         +-----------------+
```

Implementation hooks:

- **AFL++ Python custom mutator** (`afl_custom_fuzz`, `afl_custom_post_process`, `afl_custom_queue_new_entry`) — calls vLLM REST endpoint for LLM-batch mutation; `post_process` converts JSON-LLM-output → binary TLV via a hand-written serializer.
- **`afl_custom_queue_get`** filter: only LLM-mutate test cases whose terminal state is at a plateau.
- **`AFL_CUSTOM_MUTATOR_LATE_SEND`** for IPC-targeted inputs (let the orchestrator dispatch to the right node before send).
- **State tag injection**: a 256-byte preamble on each test case carries `(target_state, prior_msg_hash)` — read by `afl_custom_init_trim` to avoid trimming the script's causal prefix.

## 5. Risk register & mitigation

| Risk | Severity | Mitigation |
|---|---|---|
| CodeLlama-7B too weak for binary TLV synthesis | High | Fine-tune (LoRA) on the seed corpus + auto-generated valid traces; fall back to constrained decoding (JSON-schema → serializer). |
| kcov not exposed in container | Med | Plan B: ftrace function_graph on `softbus_server` syscalls; Plan C: user-space LD_PRELOAD shim to log every IPC. |
| Bug-yield = 0 inside 4 months | High | Pre-mine known DSoftBus CVEs (CVE-2024-…); reproduce them as "Day-0 calibration" to prove the system works before chasing 0-days. |
| LLM cost > 50 % wall time | Med | Section 3.4 scheduler; batch ≥ 16 prompts. |
| Reviewer says "this is just ChatAFL with a new target" | **Critical** | Section 3 deltas must each be ablated in the report — drop kcov, drop state-graph, drop multi-node — and show coverage regression vs the full system. |

## 6. Four-month execution plan

- **M1**: Reproduce baseline. Build DSoftBus + AFL-QEMU + run the in-tree `fuzztest/` harnesses for 7 days → branch-coverage and unique-crash numbers locked in. Static-analyze auth + transmission to draft the state graph.
- **M2**: AFL++ custom-mutator scaffold; CodeLlama-7B on vLLM; first end-to-end LLM-mutation loop on **discovery** subprotocol only (smallest state machine). Demonstrate ≥ +5 % branch on the discovery module.
- **M3**: Extend to auth + transmission; kcov fusion; multi-node orchestrator; bug-class detectors. Target full-system ≥ +15 %.
- **M4**: Triage, POCs, ablation study, technical report (with the four bug-class taxonomy table). Submit any new findings to OpenHarmony security@.

## 7. Verdict

proj22 is **a good 工程型 fit**, with a real organizer-provided container and a clean rubric. The competition risk is that "LLM + fuzzing" is now crowded enough that a baseline rerun = bottom-quartile. The five deltas in §3 are what separates a B-level award from a top-3 finish. The combo of **state-graph prompts + kcov fusion + multi-node corpus + on-device CodeLlama** is *not* what ChatAFL does and is exactly what an OS-track jury would call a systems contribution.
