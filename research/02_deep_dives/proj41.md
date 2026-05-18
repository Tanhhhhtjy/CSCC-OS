# proj41 Feasibility Report — WebAssembly Hardware-Adaptive Smart Inference Runtime

**Track:** 2026 CSCC OS Functional Track
**Base:** WasmEdge + WASI-NN, 3 vendor NPU backends (Sophon BM1684X, Rockchip RK3588, D-Robotics RDK Sunrise BPU)
**Stage 1 score:** 12/25 — re-evaluated here on the assumption boards are obtainable

---

## 1. Technical path

**Runtime pick: WasmEdge, not Wasmtime, not WAMR.** Three reasons. (a) WasmEdge already has a *production* WASI-NN plugin host with ten shipping backends (OpenVINO, TensorFlow, TFLite, PyTorch, ONNX, GGML/llama.cpp, MLX, Piper, Whisper, ChatTTS, BitNet, Burn.rs — confirmed live in `plugins/wasi_nn/` on master, Nov 2025). Wasmtime's WASI-NN is still bench-quality (one OpenVINO backend, periodic ABI churn). WAMR is geared at MCU footprints and its WASI-NN support is marked "topic only" in the README — no first-class plugin loader. (b) WasmEdge's plugin system is a C-ABI `.so` discovered by env var `WASMEDGE_PLUGIN_PATH`, which means we can ship three independent NPU `.so` files and let the *same* `.wasm` binary load whichever backend is present. That maps exactly to the topic's "one wasm, three boards" requirement. (c) The CNCF incubation status and active Chinese-vendor contributors (Second State, Intel, Sophgo collaborators on Issue #3500 about RK3588 NPU) reduce the "we picked the dead horse" risk.

**WASI-NN ABI.** The W3C spec is in **Phase 2** with a witx interface: `load`, `init_execution_context`, `set_input`, `compute`, `get_output`, plus the `graph-encoding` and `execution-target` enums (CPU/GPU/TPU/…). The vendor extension story is two-pronged: (i) register a new `graph_encoding` value (e.g. `BM1684X = 100`, `RKNN = 101`, `HORIZON_HBDNN = 102`) in the WasmEdge plugin and (ii) wire the load/compute calls through to the vendor C runtime. Tensor I/O is `(buf, dims, type)` triples — the same shape every vendor SDK already uses. **The plugin is 600–1200 LoC of C++ per backend**, modelled after `wasinn_tfl.cpp` (1.1k LoC) and `wasinn_torch.cpp` (1.3k LoC).

**Per-vendor adapter strategy.**
- **Sophon BM1684X** — link against `libsophon` (provides BMRT C API: `bmrt_load_bmodel`, `bmrt_launch_tensor`) or the higher-level **SAIL C++** (`sail::Engine`, `sail::Tensor`). Sophon ships an Ubuntu apt repo and a CMake find-module — cleanest of the three. Model format: `.bmodel` (compiled offline via `tpu-mlir`).
- **Rockchip RK3588** — link `librknnrt.so`, use the C API `rknn_init / rknn_inputs_set / rknn_run / rknn_outputs_get`. RKNN-Toolkit2 (open-source, Apache-2.0 on `airockchip/rknn-toolkit2`) ships the runtime headers, plus `rknn_model_zoo` with a working YOLOv5/v8 example. Model format: `.rknn`.
- **D-Robotics Sunrise / RDK X3/X5 BPU** — link `libdnn.so`, C API `hbDNNInitializeFromFiles / hbDNNInfer`. SDK is a closed binary blob shipped via D-Robotics's developer portal; docs are mixed Chinese/English with thin English coverage and require a developer-portal login. Model format: `.bin` (compiled offline with the OE toolchain).

**Cross-backend uniformity.** The `.wasm` guest links `wasi_ephemeral_nn` and stays vendor-agnostic. The pre-compiled model file *is* vendor-specific (we ship three model artifacts, one per backend, cross-compiled offline from the same ONNX). The plugin chooses the right C runtime via the `graph_encoding` hint at `load` time. We will publish the encoding numbers as an internal extension table and document the bijection — that table is one of our two written deliverables that has independent value beyond the demo.

## 2. AI-capability assessment (specific to this stack)

Honest per-subsystem rating, calibrated against what Claude actually knows from public docs vs. needs to be shown manually:

| Subsystem | AI proficiency (1–5) | Comment |
|---|---|---|
| WasmEdge plugin scaffolding (CMake, `wasi-nn` host module template) | **4** | Plugin source is open; Claude can produce a correct `wasinn_<vendor>.cpp/h` + `CMakeLists.txt` from the `wasinn_tfl` template with high reliability. |
| WASI-NN witx ABI / graph-encoding extension | **4** | Spec is short (≤200 lines of witx). Codegen of `set_input`/`compute` shims is mechanical. |
| Sophon SAIL / LIBSOPHON adapter | **4** | English+Chinese docs on `sophgo/sophon-sdk` and `sophgo/sophon-demo`; YOLOv5/v8 SAIL C++ examples are linkable verbatim. Claude has solid priors on `bmrt_*` from training data. |
| RKNN-Toolkit2 adapter (RK3588) | **4** | `rknn_*` C API is small (~15 functions), `rknn_model_zoo` ships a YOLOv5 C example we adapt. Claude knows this API surface. |
| D-Robotics Sunrise BPU `libdnn` adapter | **2** | Closed SDK, English docs thin, `hbDNN*` API has limited public mentions, examples gated behind login. Codegen will hallucinate. **Human must transcribe headers** from the board, then AI can finish. |
| YOLO pre/post-processing in Rust/Go/C++ guests | **5** | Trivially scaffolded. |
| COCO val2017 mAP@50 eval harness (pycocotools wrapped from wasm host) | **4** | Standard, well-documented. |
| Cross-compile toolchains (aarch64 musl, RISC-V GNU for Sophon, OE for Horizon) | **3** | Toolchain installation is finicky and AI guidance is often stale. Expect 1–2 days of human shell debugging per board. |

**Net.** Roughly **70-75% of the codegen is AI-driven** end-to-end for Sophon and RKNN paths. The Horizon path drops to ~40% AI / 60% human (board login, header transcription, OE toolchain trial-and-error). The non-AI critical path is dominated by **physical board logistics + Horizon's gated SDK**, not by coding effort.

## 3. Four-month sprint plan (8 × 2 weeks)

**Sprint 1 (W1-2) — Foundation.** Fork WasmEdge at the latest tag (currently 0.14.x family). Build `wasi_nn` with TFLite backend as the warm-up. Write a "hello-yolo" wasm guest in C++ that runs YOLOv5n on TFLite/CPU. Set up CI: x86_64 Ubuntu 22.04 + aarch64 QEMU user-mode for smoke tests.

**Sprint 2 (W3-4) — RKNN backend (target: RK3588).** Implement `wasinn_rknn.cpp` against `librknnrt`. Convert YOLOv5n ONNX → `.rknn`. Run on a real RK3588 board (Orange Pi 5 / Radxa Rock 5B — readily available, ~RMB 600). Acceptance: `.wasm` loads, runs, p50 latency ≤30ms @640×640. **RK3588 is the cheapest and best-documented board — do it first to de-risk.**

**Sprint 3 (W5-6) — Sophon backend (target: BM1684X).** Either Sophon SE7 edge box (~RMB 3000) or rented Sophon cloud node. Implement `wasinn_sophon.cpp` against SAIL C++. Convert YOLOv5n → `.bmodel` via `tpu-mlir`. Acceptance: identical guest .wasm, same demo, FPS ≥ RK3588 at INT8.

**Sprint 4 (W7-8) — Horizon backend (target: RDK X5).** **Highest-risk sprint.** RDK X5 board (~RMB 800). Transcribe `hbDNN*` headers from on-board `/opt/`, build adapter, compile YOLO via OE toolchain. If by mid-sprint the SDK is blocked, **demote Horizon to a stretch goal and substitute Cambricon MLU or a CPU-fallback path** — we explicitly call this out as the plan-B branch up front.

**Sprint 5 (W9-10) — Multi-language guests + plugin ABI hardening.** Rust guest (`wasi-nn` crate), Go guest (TinyGo + wasi-nn binding), C++ guest. Stress test: 10k inferences per backend, RSS bounded, no leaks under `valgrind`/`asan` on host plugins.

**Sprint 6 (W11-12) — COCO eval + cross-backend parity.** Run COCO val2017 (5k images) through all three backends. Report mAP@50, FPS, p99 latency, peak RSS. Build the parity table that *is the paper*.

**Sprint 7 (W13-14) — Plugin hot-swap + isolation demo.** Demonstrate that on a single host with two plugins installed, the same `.wasm` can target different `graph_encoding` values selected at runtime via an env var or a guest call. Add a fault-injection demo (kill the NPU device file, watch the wasm guest fail gracefully without crashing the runtime).

**Sprint 8 (W15-16) — Paper, demo video, defense rehearsal.** Upstream-style PR for at least one backend (RKNN is the strongest candidate — Issue #3500 is open and unowned). Final report + 10-minute demo recording across all three boards on a desk.

## 4. Skill design

Three Claude Code skills, each genuinely reusable beyond this project:

1. **`wasi-nn-backend-scaffold`** — Given a vendor name + a YAML of API symbols (`load_func`, `infer_func`, tensor I/O struct), emits `wasinn_<vendor>.cpp/.h`, the `CMakeLists.txt` block, and the `wasinntypes.h` enum addition. Wraps the WasmEdge plugin convention. Templated off `wasinn_tfl.cpp`.
2. **`vendor-sdk-adapter-template`** — Generic NPU SDK → WASI-NN bridge generator. Inputs: SDK header path, model format, tensor layout. Outputs: a tensor-marshalling layer + a smoke test that loads a known-good model and runs one inference. Designed to be re-runnable when SDKs version-bump.
3. **`coco-yolo-bench`** — Cross-backend benchmark harness. Reads three `.wasm`+model combos, runs COCO val2017, produces mAP/FPS/p99/RSS table as Markdown + CSV. Reused for the final paper.

## 5. Demo + evaluation

**Setup.** YOLOv5n (best public quantization story), COCO val2017 (5000 images, 80 classes), INT8 on all NPUs, FP32 on CPU baseline (x86_64 OpenVINO plugin).

**Target numbers we will commit to in the proposal:**

```
Backend        Board          mAP@50  FPS    p99 (ms)  Peak RSS
─────────────────────────────────────────────────────────────────
CPU/OpenVINO   x86_64 i7      0.45    18     65        180 MB
RKNN           RK3588 (3-core)0.43    72     18        95  MB
Sophon SAIL    BM1684X        0.44    140    9         140 MB
Horizon dnn    RDK X5 (10TOPS)0.42    55     22        90  MB
```

mAP drop ≤3 points vs FP32 CPU is the published quantization bound for YOLOv5n — we are not promising magic. The story is **"same wasm, three orders of magnitude in throughput across hardware tiers"**.

**Innovation claim.** The repo will contain three *upstream-quality* WASI-NN backends that **do not exist today** (confirmed by direct inspection of `plugins/wasi_nn/` on master and search of WasmEdge issues — none of Sophon/RKNN/Horizon ships). Issue #3500 is open and unowned. This is real innovation space, not "wrap what's already wrapped."

## 6. Top 3 risks + mitigations

1. **Hardware logistics (high).** Three boards across three vendors, three toolchain ecosystems, possibly three power supplies and serial cables. Mitigation: order RK3588 in W0, Sophon SE7 in W1, RDK X5 in W2 — **all three before sprint 1 ends**. Budget ~RMB 5000 total. If RDK X5 procurement slips, the plan-B branch in Sprint 4 prevents the project from blocking.
2. **Vendor toolchain breakage (medium).** Sophon `tpu-mlir`, RKNN-Toolkit2, and Horizon OE all version-bump quarterly and break ONNX→native conversion in subtle ways. Mitigation: pin one ONNX (YOLOv5n.onnx, opset 12) and one toolchain version per vendor; check `.bmodel`/`.rknn`/`.bin` artifacts into git-lfs so reviewers can re-run without re-converting.
3. **Innovation gap vs upstream WasmEdge (medium).** If during the competition window WasmEdge merges a community Sophon or RKNN backend, our novelty shrinks. Mitigation: (a) submit an early-draft PR ourselves in Sprint 7 to claim the slot; (b) lean harder on the "three vendors *unified* under one runtime + cross-language guests + parity benchmark" framing, which no single upstream PR will deliver; (c) the WASI-NN encoding-extension table itself is a deliverable independent of any single backend.

A fourth, lower-rank risk: **Horizon's closed SDK violating redistribution terms.** Mitigation — link dynamically against the on-board `libdnn.so`, ship only header shims, document the SDK install step from D-Robotics's portal.

## 7. Final pick score and rationale

**Score: 6.5 / 10** as the team's competition pick.

Rationale. Stage-1's 12/25 underrated the *innovation* axis — upstream WasmEdge genuinely has no Sophon/RKNN/Horizon backends, and the wasi-nn extension story is publication-worthy. It overrated the *vibe-coding fit* problem for Sophon and RKNN (both well-documented, English+Chinese, with open YOLO examples). It correctly flagged Horizon as a trap. Versus the shortlist: **proj61 (9.5/10) and proj18 remain stronger picks** because they live in QEMU and have zero hardware logistics. proj52/proj53 likely beat proj41 on AI-codegen leverage. proj41 is *competition-viable* if and only if the team commits to buying RK3588 + Sophon in week 0 and treats Horizon as plan-B. Otherwise it is a 4/10 trap.
