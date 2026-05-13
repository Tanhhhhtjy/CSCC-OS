# proj11 — Rust dm-crypt + dm-verity on Asterinas

Stage-2 deep dive (storage-expert, team cscc-os-stage2). Stage-1 score 23/25,
level A, engineering-type. Mentor: 田洪亮 (Ant Group / Asterinas community).
The brief asks for Rust ports of `dm-crypt` (transparent AES-XTS encryption)
and `dm-verity` (Merkle-tree integrity verification) inside Asterinas, with
FIO/SQLite throughput close to Linux and a small dm-verity boot-time overhead.

## 1. Why Trusted Storage on a Framekernel

Asterinas (USENIX ATC 2025) is a Rust framekernel: all unsafe code is confined
to OSTD, services run as safe-Rust components, and the project markets itself
as a Linux-ABI-compatible OS for Confidential Computing (CoCo). In a CoCo
deployment Asterinas is a guest kernel inside an AMD SEV-SNP / Intel TDX VM;
memory is sealed by the TEE, but the rootfs image still sits on an
*untrusted* host disk. Without on-device cryptography the host can:

- **Replace** the rootfs with a backdoored image (integrity attack).
- **Read** sensitive data at rest (confidentiality attack).
- **Roll back** to an older signed image (freshness attack — out of scope for
  this project but worth flagging).

Linux solves the first two with the Device Mapper layer: `dm-verity` chains
each data block to a SHA-256 Merkle tree whose root is verified at boot, and
`dm-crypt` does AES-XTS encryption on every sector. Reproducing both inside
Asterinas closes the storage-security gap and gives the Rust-OS ecosystem a
reusable building block — a clear A-tier engineering target.

## 2. Asterinas Block Layer — What Exists, What Doesn't

The `kernel/comps/block` crate (`aster-block`) already provides:

- A `BlockDevice` trait modeled on Linux's `request_queue`, with submission
  of `Bio` requests carrying a `BioType` (Read/Write/Flush/Discard), a
  segment list, and a per-bio completion callback.
- A `virtio-blk` front-end (under `comps/virtio`) that fully implements the
  trait; this is what QEMU benchmarks will use.
- A page-cache-like buffer-head abstraction sitting above the block trait,
  used by `aster-jinux-fs`/ext2.

What is **missing** (verified via the `comps/` listing on the GitHub main
branch — no `dm`, `device-mapper`, or stacking driver crate exists):

- A general "stacked block device" abstraction that wraps an underlying
  `BlockDevice` and exposes another `BlockDevice` to upper layers.
- A `BioContext` for asynchronous chained completions (needed because
  dm-verity has to fetch hash blocks before completing a data read).
- A device-mapper-style table mechanism for runtime composition (`crypt → verity → virtio-blk`).

This is the **single biggest project risk**: we are not just porting two
modules, we are also designing the missing abstraction underneath them.
Realistically half of the engineering budget will go into a small
`aster-dm` crate that lets stacked block devices be registered as a new
`BlockDevice` instance. Without it dm-crypt and dm-verity become point
solutions that can't be combined.

## 3. Crypto Building Blocks

| Crate            | Purpose            | Status (verified)                        |
|------------------|--------------------|------------------------------------------|
| `aes` (RustCrypto) | AES-128/256 core | Constant-time, third-party audited       |
| `xts-mode`       | AES-XTS wrapper    | Maintained, no formal audit but widely used |
| `sha2`           | SHA-256            | Part of RustCrypto, constant-time, audited |
| `subtle`         | Constant-time eq   | Audited                                   |

RustCrypto explicitly states: "only the `aes` crate provides constant-time
implementation and has received a third-party security audit." That gives us
a defensible cryptographic core. The `xts-mode` crate provides the
sector-tweakable AES-XTS construction `dm-crypt` uses (`aes-xts-plain64`
mode).

Hardware acceleration: x86 `aes` enables AES-NI automatically when
`target-feature=+aes,+sse2` is set; this is the default for `cargo osdk run
--release`. Asterinas does **not** currently expose a Kunpeng-style HW-crypto
accelerator hook; for the demo all crypto goes through AES-NI on a stock x86
host. We will report ARMv8 Crypto Extensions as future work.

## 4. dm-crypt Module Design

### 4.1 Data path
- Sector size: 4 KiB (Asterinas block layer default). XTS tweak = LBA on the
  underlying device.
- Encryption mode: **AES-256-XTS-plain64** (matches LUKS2 default).
- Key handling: master key stored in `KernKeyring` (an Asterinas-side
  in-memory keyring); never crosses the OSTD boundary into safe-Rust
  services. Zeroized on shutdown via the `zeroize` crate.

### 4.2 Bio flow
```
upper-fs ──Bio(write)──▶ aster-dm-crypt
                            │  for each segment:
                            │    in-place encrypt into a bounce buffer
                            ▼
                       underlying BlockDevice (Bio with new bounce segments)
                            │
                            ▼  (completion)
                       crypt completion: free bounces, fire upper callback
```

Reads work symmetrically: allocate bounce buffer, submit read on the
underlying device, decrypt in the completion handler, copy to the original
upper-layer pages, fire upper callback.

### 4.3 cryptsetup interop
The brief explicitly asks for `cryptsetup` compatibility. We will support
LUKS2 header parsing (read-only is enough — creation can be done on the
Linux host) so an image created via `cryptsetup luksFormat` on Linux can be
opened in Asterinas. The LUKS2 JSON header parser is small (~400 LOC) and
we can reuse `serde-json-core` (no_std).

### 4.4 Performance optimizations
- **Per-CPU bounce buffer pools** to avoid global allocator contention.
- **Vectorized 8×AES-NI** pipelining — `aes` crate already issues AESENC
  instructions on parallel blocks; we just need to feed it 128-byte chunks.
- **Optional async encrypt** offload to a kernel worker pool (`smol`-style
  task spawn) when the bio is large (≥ 64 KiB), so the submitting thread
  isn't billed for crypto CPU time.
- **No-encrypt fast-path for zero pages** during sparse file writes.

Target: ≤ 15 % throughput regression vs raw virtio-blk on 4 KiB random
read / 1 MiB sequential read FIO workloads on an NVMe-backed QEMU guest.
Linux dm-crypt with the same crypto sees ~10 % regression as a sanity floor.

## 5. dm-verity Module Design

### 5.1 Merkle tree layout
- Data block size 4 KiB, hash block size 4 KiB, SHA-256 ⇒ 128 hashes per
  hash block, fan-out 128.
- Tree stored depth-first from root, identical to Linux dm-verity so a
  `veritysetup format`-produced hash device is bit-compatible.
- Root hash + salt provided via kernel cmdline:
  `rootfs_verity.scheme=dm-verity rootfs_verity.hash=<hex>`.

### 5.2 Read path
- For each data-block read, walk root → leaf, verifying SHA-256 at each
  level. Hash blocks themselves are cached in a small LRU
  (sized at boot, default 8 MiB).
- Bio completion is **chained**: the underlying data read and the missing
  hash-block reads are submitted in parallel; the bio is completed only
  once every hash on the path matches.
- Verification failure ⇒ the bio fails with `EIO` and a kernel log line
  (`dm-verity: verification failed at lba=X`), exactly as Linux behaves.

### 5.3 Optimizations
- **`check_at_most_once`** equivalent: a per-block "verified" bitmap kept
  in RAM. First-touch verifies, subsequent reads of the same block skip
  hashing. Saves > 80 % of hash CPU on repeated reads of a hot rootfs.
- **Forward Error Correction (FEC)**: optional, RS(255,253) per Linux's
  default. Implementation can use the `reed-solomon-erasure` crate. We
  treat this as a stretch goal.
- **Boot-time prefetch**: during the early-boot rootfs mount, issue a
  background sweep that reads-and-verifies the top three levels of the
  tree into the hash-block cache. This shifts cost out of the
  user-visible critical path.

### 5.4 Boot-time overhead target
Linux dm-verity with `check_at_most_once + try_verify_in_tasklet` adds
< 100 ms to a typical 2-second boot on a small rootfs. Our target:
**≤ 200 ms additional boot time** for a 256 MiB rootfs on QEMU virtio-blk,
measured as `time-to-init` from `start_kernel` to `/sbin/init exec`.

## 6. Evaluation Plan and Risks

### 6.1 Correctness oracle
- Build a rootfs on Linux: `veritysetup format rootfs.img → root hash`.
  Mount in Asterinas using the same root hash and verify boot succeeds.
- Flip one byte in the data device → boot must fail at the corrupted block,
  with a kernel log identifying the LBA.
- Build an encrypted volume with `cryptsetup luksFormat`; open it in
  Asterinas and round-trip a 1 GiB file. Compare SHA-256 with the host.

### 6.2 Performance benchmarks (must report)
- **FIO microbench**, 4 KiB / 64 KiB / 1 MiB, random and sequential, R/W.
  Three lines per chart: raw virtio-blk, +dm-crypt, +dm-crypt+dm-verity.
  Linux runs on the same QEMU image as a reference.
- **SQLite TPC-C-lite** (filebench oltp) on top of ext2-on-dm-crypt.
  Report tx/s and p99 latency.
- **Boot time** with and without dm-verity, averaged over 20 runs.
- **Crypto CPU share** from `perf stat` — if AES-NI is missing this will
  spike and we want to detect it.

### 6.3 Targets (must-hit numbers)

| Workload                  | Linux baseline | Our floor      | Our target     |
|---------------------------|----------------|----------------|----------------|
| 4 KiB rand read +dm-crypt | -10 %          | -25 %          | -15 %          |
| 1 MiB seq write +dm-crypt | -8 %           | -20 %          | -12 %          |
| 4 KiB rand read +verity   | -5 %           | -15 %          | -8 %           |
| boot-time +dm-verity      | +80 ms         | +250 ms        | +150 ms        |
| SQLite oltp tx/s +crypt   | -12 %          | -25 %          | -15 %          |

### 6.4 Risks
1. **Missing stacking abstraction (highest risk).** No device-mapper-like
   layer currently exists in `aster-block`. We must design and land
   `aster-dm` first. Plan B: build dm-crypt as a direct filter inside the
   virtio-blk component (less reusable but unblocks the demo).
2. **Async bio chaining is non-trivial in safe Rust.** Pinning, lifetimes
   across completion callbacks, and Send/Sync of bounce buffers will
   require careful design; reuse `core::task::Waker` patterns.
3. **xts-mode crate is not formally audited.** For the competition this is
   acceptable; we document it as a known limitation and propose moving to
   a future audited XTS implementation as future work.
4. **LUKS2 header diversity.** Real cryptsetup users have many KDF
   configurations; we support only argon2i + aes-xts-plain64 + sha256
   and reject everything else with a clear error.
5. **Page-cache interaction.** Asterinas's buffer cache will see encrypted
   bytes from dm-crypt and verified bytes from dm-verity; we need to make
   sure both layers play nicely with cached reads (the hash needs to be
   recomputed only on bio submission, not on cache hits).
6. **TEE attestation linkage** (out of scope but worth a paragraph in the
   final report): the root hash should ideally be sealed to a measured
   TEE state to defeat rollback, e.g. via a UEFI variable or vTPM PCR.
   We will sketch but not implement.

### 6.5 Project layout

```
asterinas-fork/
├── kernel/comps/
│   ├── dm/           # new: stacking abstraction, table parser
│   ├── dm-crypt/     # new: aes-xts pipeline, LUKS2 parser
│   ├── dm-verity/    # new: SHA-256 merkle tree, FEC
│   └── block/        # extended: BioContext, chained completions
├── osdk/test/        # FIO/SQLite/boot benches
└── docs/             # design doc + user manual (Chinese + English)
```

LOC estimate: ~2 500 Rust for dm-crypt, ~2 000 for dm-verity, ~1 500 for
`aster-dm` plumbing, ~800 for LUKS2 header parsing, ~1 000 for tests and
benches. A 4-person team across one semester is feasible if the stacking
abstraction is prototyped in the first two weeks.

## Conclusion

proj11 is a high-value engineering task with a clean separation of
concerns: well-understood Linux references, mature Rust crypto libraries,
and a clear correctness oracle (Linux `cryptsetup`/`veritysetup`). The
non-obvious work is the device-mapper-like stacking layer that Asterinas
currently lacks — calling that out early and landing it first is the
difference between a polished A-grade submission and a partial demo.
