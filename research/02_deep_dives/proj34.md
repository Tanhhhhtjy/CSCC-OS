# Stage 2 Deep Dive — proj34: xv6 VFS + HAL (RISC-V + LoongArch)

- Topic code: proj34 (items[33])
- Track: 2026 全国大学生计算机系统能力大赛 - OS 功能挑战赛道
- Level: B · Tags: 教学型 · Tutor: 文艳军 (NUDT)
- Stage 1 score: 23/25
- Target platform: qemu-system-riscv64 + qemu-system-loongarch64
- Deliverables (per `target` field): 设计精巧度 20 %, 功能实现 20 %, 稳定性 20 %, 文档 20 %, Demo 20 %
- Hard requirements (per `topicreq`): xv6FS + ext2 + fat32 simultaneously; RISC-V + LoongArch; pass xv6-riscv rev5 self-tests on both; pass `xv6-extend-vfs/tests` (yanjun-wen, the tutor); submit final diff vs xv6-riscv rev5.

## 1. Why this is a "comfortable but crowded" pick

Stage 1 already flagged that **two previous contest winners** exist (静春山 / RuOK, both 2025 内核赛道 全国赛). The tutor (文艳军 / NUDT) is also the maintainer of the canonical `xv6-extend-vfs` test repo. Combined this means:

- The expected solution shape is **known** — design lattice will be a Linux-style `struct file_operations` + `struct inode_operations` + `struct super_block` + per-fs driver, mounted into a path-resolution table.
- The judge already has reference solutions and a curated test harness. Your delta is measured against them.
- Reviewers know what "clever" looks like and what "obviously bolted on" looks like. **Code-elegance / diff-size is explicitly 20 %** ("设计精巧度: 根据文档和diff文件度量"). A 12 000-line patch will lose to a 4 000-line patch even if both pass.
- The tutor's own test suite (verified via WebSearch: `tests/cmds.sh`, `hello.c`, `test1.c`, `test2.c` only) is *minimal*. Passing it is necessary but not sufficient; differentiation must come from beyond.

## 2. State of the art (verified May 2026)

- **xv6-riscv rev5** (MIT, `mit-pdos/xv6-riscv` tag `xv6-riscv-rev5`): single-FS, RISC-V-only, ~9 kLoC. The baseline for the diff metric.
- **xv6-rust** (`Ko-oK-OS/xv6-rust`, WebSearch verified): does *not* introduce a vnode layer; instead enums + Arc for FileInner. **Not** a VFS reference for our purposes, but a useful illustration of why a naive Rust port fails to factor the FS — we want to do *better* than that.
- **xv6-extend-vfs** (`yanjun-wen/xv6-extend-vfs`, tutor's repo): "扩充 xv6-riscv 第5版，增加 VFS"; sample boot uses xv6 FS at `/`, manually `mount` ext2 onto `/mnt`. QEMU command pinned: `qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 1 -nographic -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0`.
- **静春山 / RuOK 2025 repos**: gitlab.eduxiji.net IDs `T202510558995330-264` and `T202510486995232-2402`. Project pages return only metadata, not source; team must `git clone` both at S0 to see their architecture. Action item: extract their diff size, VFS interface signatures, and HAL split boundary.
- **LoongArch xv6 prior art**: no canonical public port surfaced in search. The team must build the LoongArch side largely fresh, using OpenSBI/LoongArch UEFI documentation + qemu-system-loongarch64 `virt` board + Loongson-3A5000-style ISA references. This is the **highest technical risk** in the project.

## 3. Architectural sketch

Two orthogonal abstractions; design them in parallel but commit them separately for a clean diff story.

### 3.1 VFS layer (`kernel/vfs/`)

Adopt a thin Linux-flavoured contract:

```c
struct super_block {
    const struct super_operations *s_ops;
    struct fs_type *s_fs;     // back-pointer to type ("xv6","ext2","fat32")
    void           *s_priv;   // per-fs root data
    struct vnode   *s_root;
};
struct vnode {                // VFS-level inode; "inode" in xv6 collides
    uint32_t v_ino;
    uint16_t v_mode;
    uint32_t v_nlink;
    uint64_t v_size;
    const struct vnode_operations *v_ops;
    const struct file_operations  *v_fops;
    struct super_block *v_sb;
    void   *v_priv;           // fs-specific inode (xv6 dinode, ext2 inode, fat dir-entry)
    struct vnode *v_mountpoint; // non-NULL if a sub-mount lives here
};
struct vnode_operations {
    int (*lookup)(struct vnode*, const char*, struct vnode**);
    int (*create)(struct vnode*, const char*, uint16_t, struct vnode**);
    int (*unlink)(struct vnode*, const char*);
    int (*mkdir)(struct vnode*, const char*, uint16_t);
    int (*rmdir)(struct vnode*, const char*);
    int (*readdir)(struct vnode*, struct dirent*, uint64_t off);
    int (*truncate)(struct vnode*, uint64_t);
    int (*getattr)(struct vnode*, struct stat*);
    int (*sync)(struct vnode*);
};
struct file_operations {
    ssize_t (*read)(struct file*, char*, size_t, uint64_t*);
    ssize_t (*write)(struct file*, const char*, size_t, uint64_t*);
    int     (*ioctl)(struct file*, uint64_t, void*);
    int     (*close)(struct file*);
};
struct super_operations {
    int (*mount)(struct super_block*, struct device*, const char*opts);
    int (*umount)(struct super_block*);
    int (*sync)(struct super_block*);
    int (*statfs)(struct super_block*, struct statfs*);
};
```

Path resolution becomes a single `namei()` walker over vnodes, with a single `struct mount` table mapping mountpoint vnodes to child superblocks. The existing xv6 `iget`/`iput`/`ilock` becomes private to the xv6FS driver and is renamed (`xv6_iget` etc.) to avoid namespace collision with VFS.

### 3.2 HAL (`kernel/hal/<arch>/`)

Hide every privileged-instruction access behind 30-or-so functions. Concretely:

- Boot: `hal_boot_entry()` ; `hal_setup_pagetable()` ; `hal_jump_to_main()`
- CPU: `hal_cpu_id()`, `hal_intr_on/off`, `hal_wfi`, `hal_dsb`, `hal_isb`
- Trap: `hal_trap_init()`, `hal_kernelvec`, `hal_uservec`, `hal_usertrap_ret()` — abstracts SATP/SSTATUS (RV) vs CRMD/PRMD/ESTAT/EUEN (LoongArch).
- MMU: `hal_pte_t`, `hal_pte_flags`, `hal_walk()`, `hal_map_page()`, `hal_flush_tlb_one/all()` — abstracts Sv39 vs LA64's 4-level page table format.
- Timer: `hal_timer_init`, `hal_timer_set_next`, `hal_timer_now_ticks` — RV uses CLINT mtimecmp, LA64 uses CSR TCFG.
- UART: `hal_uart_putc/getc` — NS16550 on both architectures (QEMU virt boards), so single driver shared.
- VirtIO MMIO transport: identical on both boards (`virtio-mmio` on RV `virt`, `virtio-pci` on LoongArch `virt`). Must abstract or pick MMIO consistently. Choosing MMIO-only on both keeps the disk driver shared.

The HAL should be **header-only contract + per-arch C file**; do not introduce a runtime dispatch table. The diff-size grader will subtract any function-pointer indirection the compiler can't inline away.

## 4. Differentiation against the two prior winners

Since both 静春山 and RuOK already passed the same tutor's tests in 2025, the team must out-perform them on the 100 - 20 = 80 % of grade that isn't pure correctness. Strategies:

1. **Smaller, prettier diff**. Target ≤ 4 500 added lines (rev5 baseline is ~9 000; both winners likely added 8–15 kLoC). Achieved by: (a) header-only HAL contract, (b) reuse xv6's existing block-buffer + log layer underneath ext2/fat32, (c) compile out unused FS drivers at config time.
2. **A third filesystem, optional**. The brief mandates xv6+ext2+fat32. Add a fourth tiny one as stretch — e.g. `tmpfs` (RAM-backed, ~200 LoC) — to demonstrate the VFS *is* a real interface, not a 2-way switch hiding behind one. Or implement `procfs` exposing `/proc/<pid>/{status,cmdline}` for demo polish.
3. **Crash-recovery test rig**. Inject random faults during disk writes (qemu monitor `quit` or a custom block-device wrapper that drops writes after N blocks); rerun fsck-equivalent; assert FS still mountable. xv6's log layer makes this *almost free* for xv6FS; document ext2 lack of journaling as a known caveat; for fat32 implement scandisk-equivalent. This rig is *missing* from both 2025 reference projects and is high-value for the "稳定性" 20 %.
4. **Microbenchmark vs Linux ext2** on identical QEMU image. Compare seq-read, rand-read, dir-walk, `dd` throughput. Even a "we're 0.7x of Linux at sequential read on 128 MiB image" plot tells reviewers we're serious. Use `fio`-style minimal user-space benchmark (we don't have libc; write a 200 LoC `bench.c` user program).
5. **Same kernel binary, two architectures**. Make `make ARCH=riscv` and `make ARCH=loongarch` produce two flavours from one tree with **zero duplicated C files outside `hal/`**. Both prior winners probably split trees more aggressively; we win on elegance metric.
6. **Reproducible demo**. A single script `make demo` boots RV + LA, mounts ext2 + fat32 + xv6fs, copies a file across all three, runs all `tests/`, prints PASS table. Reviewers love one-button demos.

## 5. Test plan and pass criteria

| Layer | Tool | What it checks |
|-------|------|----------------|
| Baseline | xv6 rev5 `usertests` | Regression: no xv6 feature broken. Both archs. |
| Tutor tests | `xv6-extend-vfs/tests/{hello,test1,test2,cmds.sh}` | Hard pass gate. |
| Filesystem | `mkfs.ext2`+`mkfs.fat` from host, mount in xv6 | Cross-tool interop: file created in host's ext2 image is readable in xv6, and vice versa. |
| Crash | Custom `crash-blkdev` shim | After N forced power-cuts, FS remountable, no torn metadata for xv6FS; ext2 marked dirty + minimal fsck-equivalent. |
| Perf | Custom `bench.c` | Numbers reported in final doc; no minimum threshold required. |
| Arch parity | Same testset on RV + LA | All tests pass on both. |

## 6. Sprint plan (16 weeks, 1 lead + 2 engineers; tracks proj9 timeline)

| Sprint | Weeks | Output |
|--------|-------|--------|
| S0 | 1 | Clone xv6-riscv rev5, 静春山, RuOK, xv6-extend-vfs; reproduce all four builds; tabulate their VFS interface diffs. |
| S1 | 2–3 | Insert VFS layer above existing xv6FS (no new FS yet). Tutor tests pass on RV; baseline `usertests` green. Diff ≤ 600 LoC. |
| S2 | 4–5 | ext2 driver, read-only first then write. Cross-mount test against host-built ext2 image. |
| S3 | 6–7 | fat32 driver (short-name only; long-name optional). Cross-mount with mtools-built image. |
| S4 | 8 | HAL refactor for RISC-V (no behaviour change). Diff localised under `hal/`. |
| S5 | 9–10 | LoongArch port: boot from QEMU virt, UART, MMU, timer, trap. Reach init `sh` prompt. |
| S6 | 11 | LoongArch VirtIO MMIO disk + all three FS drivers running. Full test pass on LA. |
| S7 | 12 | Crash-recovery rig; stability soak (24 h forced random reset loop). |
| S8 | 13 | Benchmarks vs Linux ext2 inside same QEMU. Stretch: tmpfs / procfs driver. |
| S9 | 14–15 | Documentation (design report incl. annotated diff), demo script, video. |
| S10 | 16 | Buffer / freeze. |

## 7. Risk register

- **LoongArch boot**: largest single risk. Mitigation: spend S0 day 1 on `qemu-system-loongarch64 -machine virt -bios <opensbi-or-uefi>` smoke-test of a 50-line hello-kernel; if blocked > 1 week, fall back to "RISC-V plus a *partial* LoongArch port behind a feature flag" and renegotiate scoring weights via the tutor early.
- **fat32 long file names**: VFAT LFN is patent-encumbered historically; many teaching kernels stop at 8.3. Document the limitation; not a points hit if explicit.
- **ext2 journaling absent**: ext2 has no journal by spec. Don't try; ship ext2 read/write + dirty-flag + scandisk-style fsck-light.
- **Diff bloat**: hard cap of 4 500 LoC. Code-review every PR for "can this live in HAL instead of being duplicated?".
- **Reference solutions outpace ours**: track 静春山/RuOK styles; do not copy code (the diff comparison may include similarity hashing).

## 8. Differentiation summary (what to tell judges)

> "Two existing winning entries solved this; we did it in **half the lines** by enforcing a header-only HAL contract, **on both QEMU virt boards** with one source tree, with a **fault-injection rig** that proves stability and a **fourth filesystem** that confirms the VFS contract is genuine. Plus head-to-head benchmark numbers against Linux ext2 on the same QEMU image."

That sentence maps cleanly onto 20 %+20 %+20 %+20 %+20 % = each of the five scoring axes is independently addressed, and the "elegance" / "stability" axes — where prior winners are weakest by absence — become our edge.

## 9. Bottom line

proj34 is the safest 23/25 in the field, but the price of safety is crowding. The plan above turns the crowding into an advantage: we explicitly out-engineer the public 2025 winners on dimensions (diff size, dual-arch unification, crash recovery, benchmark) where teaching-kernel entries usually punt. Combined with proj9 (high-ceiling Rust-OS engineering) it forms a balanced two-track entry portfolio for the 2026 contest cycle.
