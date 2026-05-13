# proj6 — eBPF Network Service Acceleration & Dual-End Cache

Topic code: `proj6` (level B, 学术型). Tutor: 万胜刚 (HUST). Stage-1 promise score: 24/25. This report drills into the technical path, the four-month sprint, supporting Claude Code skills, demo/evaluation methodology, the top three risks with mitigations, and a final pick score.

## 1. Restated problem & scope discipline

The brief asks for three concurrent capabilities:

1. eBPF-based monitoring **and** acceleration for ≥2 network services drawn from `{DNS, gRPC, LDAP, Redis, SMB, …}`.
2. KVM host-NIC ↔ guest-app fast path (intercept at host NIC, short-circuit or path-optimize toward the VM).
3. A **dual-end cache** (client side + server side) whose policy/algorithm is configurable at runtime.

The "学术型" tag and 25% innovation weight push us away from a pure libbpf reproduction. We want a defensible architectural claim that an existing tool (ONCache, Cilium L7, sslsniff) does not already deliver.

## 2. Opinionated protocol pick: **DNS + Redis (RESP), not DNS + gRPC**

The reference data and most public eBPF L7 work assume the team will pick DNS + gRPC. We argue the **better pair is DNS over UDP/53 and Redis RESP over TCP/6379**, because:

* **Parsing tractability inside the verifier.** DNS query names are length-prefixed labels (≤63 bytes per label, ≤255 total) and the question section is at a fixed offset — feasible with a bounded loop and `bpf_loop()` helper (kernel ≥5.17). RESP is even simpler: ASCII line-oriented (`*N\r\n$len\r\n…`). Both fit in a few hundred verifier insns. **gRPC sits on HTTP/2**, which means HPACK header compression with dynamic state per stream, frame fragmentation, and TLS in the realistic case. Parsing HTTP/2 inside an sk_msg / sockops program is widely known to be the hard ceiling that even Cilium's L7 policy delegates to a userspace Envoy via a redirect — not an in-kernel cache. Picking gRPC forces us into either (a) a userspace proxy (loses the eBPF acceleration story) or (b) a degenerate "TLS off, prior-knowledge h2c, fixed schema" demo that the judges will see through.
* **Cache value proposition is realer.** Redis is *the* canonical caching protocol; a transparent client+server kernel-resident cache for GET/SET hot keys is an honest performance win (hit-path skips userspace round-trip entirely via `bpf_msg_redirect_hash`). DNS caching is the textbook eBPF acceleration story (Cloudflare, dnsdist sk_lookup work). Both have published baselines.
* **Failure modes are bounded.** Cache miss falls back to normal socket; nothing to break. With gRPC, a misparse on HPACK state can corrupt a stream silently.

We retain LDAP/SMB as **stretch** protocols (LDAPv3 BER is harder than RESP but tractable; SMB2 has clean fixed headers). We do **not** commit to gRPC.

## 3. Architecture

```
┌──────────────────── Userspace control plane (Rust, aya) ───────────────────┐
│  cache-policy daemon: LRU / TinyLFU / W-TinyLFU; TTL config; map snapshot │
│  metrics exporter (Prometheus) | CLI (clap)                               │
└────────────┬──────────────────────────────┬───────────────────────────────┘
             │ pinned maps, ring buffers    │
┌────────────▼──────────────────────────────▼───────────────────────────────┐
│ Kernel eBPF programs                                                       │
│  • DNS:  XDP on host NIC + tc-egress on veth; cache map (qname,qtype)→A/AAAA│
│  • RESP: sockops (capture connect/accept) → sk_msg verdict → hash redirect │
│          cache map (db,key) → value blob in BPF_MAP_TYPE_HASH (or lru_hash)│
│  • KVM:  XDP_REDIRECT on host NIC into tap/macvtap with cpumap fanout;     │
│          AF_XDP zero-copy ring for outliers; bpf_redirect_peer for veth    │
└────────────────────────────────────────────────────────────────────────────┘
```

**Why aya (Rust) over libbpf-C.** Aya is mature enough for sk_msg/sockops/uprobe/XDP/kprobe and gives us safer userspace control logic for the cache daemon. We keep a libbpf-C fallback path for any program type where aya lags (e.g., LSM, sleepable kprobes).

**Why a kernel patch is avoided.** ONCache (NSDI'25) ships an rpeer patch (`bpf_redirect_rpeer`). For a contest we cannot ask judges to patch their kernel. The mainline `bpf_redirect_peer` (kernel 5.10+) is sufficient for the host↔veth case; for tap/macvtap to KVM guests we use XDP_REDIRECT into the tap fd or AF_XDP. This is a deliberate scope cut to keep the demo bootable.

**Why dual-end.** Client-side cache (in the application's veth ingress) collapses the round-trip for popular queries. Server-side cache (on the server netns egress) collapses repeated workloads from different clients. The interesting research question — and our innovation hook — is **invalidation coordination**: server-side SET/DEL events broadcast invalidations via a `BPF_MAP_TYPE_PERCPU_ARRAY`-backed eventfd to clients in the same host; cross-host uses a small userspace gossip on UDP.

## 4. Four-month sprint plan

| Month | Milestone | Deliverable | Gating exit criterion |
| --- | --- | --- | --- |
| M1 (Mar) | Skeleton + DNS path | aya project compiles; XDP+tc DNS cache works on host netns; libbpf-bootstrap-style scaffold. | `dig` against a stub authoritative server shows ≥80% cache hit latency in single-digit μs vs ~200μs uncached. |
| M2 (Apr) | Redis RESP + sockops/sk_msg redirect | Two redis-cli clients on the same host hit a cached GET without touching the server process. Cache policy daemon supports LRU + TinyLFU at runtime. | YCSB-A workload (50/50 GET/SET) on `localhost` shows ≥30% throughput uplift on read-heavy mix. |
| M3 (May) | KVM fast path + dual-end coordination | XDP_REDIRECT host NIC → tap into a Debian KVM guest running redis-server; invalidation gossip working across two hosts via DPDK-free UDP. | Inter-VM ping RTT reduced ≥20%; redis-benchmark across host↔guest hits 80% of bare-metal QPS. |
| M4 (Jun) | Hardening + evaluation + docs | Stress + verifier coverage; CO-RE build; Prometheus dashboards; full test matrix (kernel 5.15/6.1/6.8); writeup. | All targets in §5 met; CI green on the three kernels; 20-page report + 6-minute demo video. |

Each month ends with a tagged release. The four-month budget assumes two ~full-time team members; with one member, drop the cross-host gossip (it is the lowest-value bit) and demo only single-host dual-end.

## 5. Demo & evaluation methodology

* **Workloads.**
  * DNS: a synthetic Zipfian qname trace (α=1.0, 1k unique names, 1M queries) via `dnsperf`.
  * Redis: YCSB-A/B/C on a 1M-key dataset, value size 64B/1KB/16KB, against a real `redis-server` 7.x in a KVM guest.
  * KVM: `iperf3`/`netperf TCP_RR` host→guest baseline vs eBPF fast path.
* **Metrics.**
  * Hit ratio (per-protocol), median + p99 latency, throughput, syscall count (perf-stat), %CPU delta (ours vs baseline, same workload).
  * **Overhead budget**: cache disabled must add ≤2% CPU vs vanilla; cache enabled must not regress *any* metric on the cold-miss path.
* **Correctness.**
  * Differential testing: every cached response is compared (offline, on a sampled tap) against an uncached oracle. Zero divergence is the bar.
  * RFC 1035 DNS edge cases (compressed names, EDNS0 OPT, truncation), RESP pipelining, RESP3 push messages.
* **Reproducibility.** A `make demo` target spins up a vagrant/qemu KVM guest, loads programs, runs the trace, prints a table.

## 6. Claude Code skills (custom + reused)

We will lean on Claude Code as a co-developer for verifier wrangling and protocol parsers. Concrete skills:

1. **`ebpf-verifier-doctor` (custom skill).** Input: a failing `clang -O2 -target bpf` build log or `bpftool prog load` strace. Output: ranked hypotheses (loop bound, stack >512B, pointer arithmetic without bounds check, helper-not-allowed-here) with the exact source-line edits. Backed by a corpus of verifier error strings from kernel `verifier.c` and known fixes from libbpf-bootstrap issues.
2. **`resp-dns-parser-synth` (custom skill).** Generates verifier-safe parsers for line-oriented and length-prefixed binary protocols. Templates RESP/DNS/SMB2 with explicit `bpf_loop` bounds and bounds-checking macros.
3. **Reused: `superpowers:test-driven-development`.** Each parser ships with a userspace fuzz harness *before* the kernel program; only after the userspace version passes 1M libfuzzer iterations does it move into BPF.
4. **Reused: `superpowers:systematic-debugging`.** For the inevitable "works in userspace, dies in verifier" gap.
5. **Reused: `simplify`.** Runs after each milestone to dedupe map definitions and tighten the daemon.

The two custom skills are genuinely novel here because the existing `eunomia-bpf` toolchain is great for boilerplate but does not coach the developer through verifier failures or generate protocol-aware parsers. Both ship as small `.claude/skills/*.md` files plus a Python helper that consumes the kernel error.

## 7. Top 3 risks and mitigations

1. **Verifier rejections on the RESP cache write path.** Storing variable-length values in a BPF map requires either fixed-size slots (wastes memory) or a chained-chunk design (more pointer math, more verifier pain). *Mitigation:* cap value-size cache at 4KB (covers 90%+ of Redis traffic per Twitter/Memcache traces); values >4KB pass through uncached. Use `BPF_MAP_TYPE_LRU_HASH` with a fixed-size value struct `{u32 len; u8 buf[4096];}`. Validate on kernel 5.15 (oldest target).
2. **KVM packet-path depth.** XDP_REDIRECT into tap is supported but quirky; macvtap variants differ; vhost-net offload can fight us. *Mitigation:* commit to one virtual NIC backend (vhost-net with tap) for the demo, document the assumption, leverage `bpf_redirect_peer` for veth-based microvm scenarios as the secondary path. AF_XDP is the safety net.
3. **Innovation gap vs ONCache/Cilium.** Judges may say "this is ONCache minus a kernel patch." *Mitigation:* the **dual-end coordinated invalidation** for Redis is genuinely unaddressed in ONCache (overlay net only) and Cilium (no app-level cache). We frame the contribution around that, not around overlay throughput.

## 8. Cross-references to the ecosystem (verified)

* **ONCache** (github.com/shengkai16/ONCache, NSDI'25): TCP overlay fast path, requires kernel patch for rpeer, ~37 Gbit/s vs 39.5 bare-metal. Useful as a *baseline* and as a source of the TC-based overlay short-circuit pattern; we do **not** depend on its rpeer patch.
* **Cilium sockops/sk_msg**: production reference for `bpf_msg_redirect_hash` to short-circuit local socket pairs. We will reuse the pattern; the bpf/sockops tree URL changes across releases (the path 404s today on main), so we pin to a release tag.
* **bpftime** (eunomia-bpf/bpftime): userspace eBPF runtime; **not used** for proj6 because the value of proj6 is in-kernel acceleration. Noted for proj32.
* **eunomia-bpf**: 90% Rust, CO-RE + Wasm distribution. We adopt its CO-RE build pattern but not its Wasm runtime.
* **aya-rs**: 4.5k stars, supports the program types we need (verified for kprobe/uprobe/XDP/tc/sockops/sk_msg via the aya-rs/aya-examples repo, even though the upstream README only highlights CgroupSkb).
* **libbpf-bootstrap**: ships uprobe/kprobe/xdp/tc/sockfilter examples; **no sk_msg/sockops example** — we will contribute one back if upstream-worthy.

## 9. Pick score: **7.5 / 10**

Rationale:

* (+) Mature ecosystem; verifiable baselines (ONCache, dnsdist); demo is naturally visual.
* (+) The dual-end invalidation angle is a real research delta.
* (+) Tooling (aya, libbpf-bootstrap) is contest-friendly.
* (−) Level B (not Level A), so headroom for top-tier prize is structurally lower than proj32.
* (−) Three sub-features (≥2 protocols, KVM, dual-end) is a lot of surface; finishing 2.5 of them well beats finishing 3 poorly.
* (−) Heavy competition: every cohort fields an "eBPF accelerator" team; differentiation matters and is not automatic.

Recommendation: pick proj6 if the team is strong on kernel/networking and weaker on ML/agent semantics. Otherwise prefer proj32.
