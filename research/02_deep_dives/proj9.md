# Stage 2 Deep Dive — proj9: Asterinas ptrace / Process Debugging

- Topic code: proj9 (items[8])
- Track: 2026 全国大学生计算机系统能力大赛 - OS 功能挑战赛道
- Level: A · Tags: 工程型 · Tutor: 田洪亮 (星绽社区 / Ant Group)
- Stage 1 score: 23/25
- Target platform: x86-64 QEMU, Asterinas Framekernel
- Deliverables (per `target` field): 50 % implementation completeness, 20 % docs, 20 % demo, 10 % innovation
- Hard requirement: GDB / strace usable against Asterinas user processes; LTP + kselftest ptrace cases pass.

## 1. Threat-of-collision check (CRITICAL UPDATE)

A WebSearch sweep of `github.com/asterinas/asterinas` PRs (May 2026) shows the upstream community is **already deep into ptrace work**:

| PR | Title | Status (May 2026) |
|----|-------|------|
| #2984 | Add `ptrace` syscall | Merged 2026-04-26 |
| #3018 | Add Yama ptrace scope | Merged 2026-03-19 |
| #3061 | Support `PTRACE_SETOPTIONS` | Open, in progress |
| #3065 | Support debugging with `ptrace` | Open, in progress |
| #3078 | Initial LSM framework + migrate Yama | Open |
| #3131 | Capability LSM module migration | Open |
| #3203 | Check `has_pending` before entering user mode | Open (signal-stop plumbing) |

Implication: **the "implement ptrace from scratch" framing is already obsolete by the time the contest opens.** Upstream will plausibly land basic attach/detach + PEEK/POKE + CONT + SETOPTIONS before our project window starts. A naive re-do scores poorly on the 10 % innovation axis and risks "merely re-implementing what is already upstream" reviewer comments. The team **must reframe** as one of:

1. **Catch-up + finish**: track upstream branch, finish the long tail (PTRACE_SYSCALL, PTRACE_GETREGSET / NT_PRSTATUS, signal-stop + wait() interplay, thread-group tracing, ptrace-stop-during-execve, SECCOMP_GET_FILTER), get LTP/kselftest **green**, drive GDB hardware-style watchpoints via DR0–DR3 surfaced through PTRACE_POKEUSER. This is a high-value contribution because PR #3065 is far from full LTP coverage.
2. **Verification & safety angle**: build a property-based test rig (proptest + ptrace state machine model) that proves the Asterinas ptrace implementation matches Linux's observable semantics on a randomised trace; ship the rig as a long-lived gift to the community. This plays directly to the title-task's "explore Rust-OS-specific debugging" innovation hook.
3. **Type-safe tracer API**: expose a *second*, Rust-native ptrace facade (capability-typed, no `unsafe`, no foot-guns like double-attach) **alongside** the Linux-ABI ptrace, and demonstrate a small in-tree debugger (`asterdbg`) on top of it. The 10 % innovation axis directly rewards this.

Recommendation: pursue **(1) + (3)** in parallel — (1) is the safe 70-point floor; (3) is the differentiating ceiling. (2) becomes a free by-product of (1) since the proptest harness *is* the test suite.

## 2. Asterinas process/thread model maturity (May 2026)

From the repo top page: "230+ Linux system calls supported", "production-ready intent". From open-PR scan:

- Signal delivery refactor still active (#3203). Important because ptrace signal-stop hangs off signal-deliver-stop.
- IPv6 (#3129), capability LSM (#3131), sendfile fixes (#3191) — not blockers but show that even basic Linux semantics are still landing.
- Thread / thread-group abstractions appear to exist (Yama operates on them) but are *not yet exhaustively tested*. Yama (#3018) was merged before ptrace itself was merged, so the policy hook is in place — we inherit it.

Risk register the team must watch:
- `wait4()` + `WSTOPPED` integration with ptrace-stop must be re-audited; this is where most LTP failures will originate.
- `execve()` ptrace event: PTRACE_EVENT_EXEC and PTRACE_O_TRACEEXEC require coordinating with Asterinas's exec implementation; verify it preserves the tracer after argv/envp/aux replacement.
- `clone()` traps for PTRACE_EVENT_CLONE/FORK/VFORK depend on the clone-flags plumbing. PR #3203 ("has_pending") hints that pre-user-mode entry checks are still being shaken out.
- LoongArch test infra is already covered by PR #3120 — but the contest mandates x86-64 only for proj9, so we keep test matrix narrow.

## 3. PTRACE_* request dependency graph and sprint plan

The ~40 PTRACE_* requests in Linux man page section 2 fall into seven dependency tiers. Implementing in this order means each tier can be tested independently and unlocks the next.

```
Tier 0 (infra, no observable request)
  ptrace stop state machine, tracer/tracee link in task_struct, Yama scope check
  wait()/wait4()/waitpid() integration: WSTOPPED + __WALL semantics
  attach permission via Yama (#3018 already merged)

Tier 1 — Attach / Detach (independent, must come first)
  PTRACE_TRACEME
  PTRACE_ATTACH        (sends SIGSTOP, induces signal-stop)
  PTRACE_SEIZE         (no SIGSTOP, modern attach)
  PTRACE_DETACH
  PTRACE_INTERRUPT     (used with SEIZE)
  PTRACE_KILL          (deprecated but tested by LTP)

Tier 2 — Memory peek/poke (read-only first, then write)
  PTRACE_PEEKDATA / PTRACE_PEEKTEXT
  PTRACE_POKEDATA / PTRACE_POKETEXT
  PTRACE_PEEKUSER
  PTRACE_POKEUSER      (DR0..DR7 for HW breakpoints — gates GDB watchpoints)

Tier 3 — Register access (no semantics dependency on memory, easy to land)
  PTRACE_GETREGS / PTRACE_SETREGS         (x86_64 user_regs_struct)
  PTRACE_GETFPREGS / PTRACE_SETFPREGS
  PTRACE_GETREGSET / PTRACE_SETREGSET     (NT_PRSTATUS, NT_FPREGSET, NT_X86_XSTATE)

Tier 4 — Execution control (requires tier 1+3)
  PTRACE_CONT          (with signal injection)
  PTRACE_SINGLESTEP    (TF flag in RFLAGS — interacts with x86 trap-flag plumbing)
  PTRACE_LISTEN        (paired with SEIZE/INTERRUPT)
  PTRACE_SYSEMU        (skip syscall — needed by user-mode emulators, low priority)
  PTRACE_SYSEMU_SINGLESTEP

Tier 5 — Syscall tracing (this is where strace lives)
  PTRACE_SYSCALL
  syscall-enter-stop / syscall-exit-stop bookkeeping (TIF_SYSCALL_TRACE bit)
  PTRACE_GET_SYSCALL_INFO   (kselftest get_syscall_info.c)
  PTRACE_SET_SYSCALL_INFO   (kselftest set_syscall_info.c)

Tier 6 — Options + events + clone-following
  PTRACE_SETOPTIONS / PTRACE_GETOPTIONS    (PR #3061 already in flight)
    PTRACE_O_TRACESYSGOOD
    PTRACE_O_TRACEFORK / TRACEVFORK / TRACECLONE
    PTRACE_O_TRACEEXEC / TRACEEXIT / TRACEVFORKDONE
    PTRACE_O_TRACESECCOMP / EXITKILL / SUSPEND_SECCOMP
  PTRACE_GETEVENTMSG
  PTRACE_GETSIGINFO / PTRACE_SETSIGINFO
  PTRACE_PEEKSIGINFO     (kselftest peeksiginfo.c)
  PTRACE_GETSIGMASK / PTRACE_SETSIGMASK
  PTRACE_LISTEN (group-stop pause)

Tier 7 — Advanced / x86-specific (optional, push to stretch)
  PTRACE_GET_THREAD_AREA / PTRACE_SET_THREAD_AREA  (32-bit; can skip)
  PTRACE_ARCH_PRCTL                                 (FSBASE/GSBASE — strace cares)
  PTRACE_SECCOMP_GET_FILTER                          (only if seccomp lands)
  Hardware breakpoint via PTRACE_POKEUSER on DR regs (already at tier 2)
```

### Sprint plan (4-month window, 1 lead + 2 engineers)

| Sprint | Weeks | Goal | Test gate |
|--------|-------|------|-----------|
| S0 | 1 | Track upstream; rebase onto PR #3065 + #3061 branches; reproduce existing tests. | Existing Asterinas regression suite green. |
| S1 | 2–3 | Finish Tier 0 + Tier 1. Fix wait()/WSTOPPED edge cases. | LTP ptrace01 + ptrace05 pass. |
| S2 | 4–5 | Tier 2 + Tier 3. Memory + register R/W. | kselftest `vmaccess.c` pass; strace can print register dumps via GDB stub. |
| S3 | 6–7 | Tier 4. CONT, SINGLESTEP, signal injection. | LTP ptrace07; GDB `stepi`/`continue`/`break` works on a hello-world. |
| S4 | 8–9 | Tier 5. PTRACE_SYSCALL + GET_SYSCALL_INFO. | `strace ls` produces correct syscall list on Asterinas; kselftest `get_syscall_info.c` + `set_syscall_info.c` pass. |
| S5 | 10–11 | Tier 6. SETOPTIONS + clone-following + PEEKSIGINFO + GETSIGINFO. | LTP ptrace02, ptrace04, ptrace08, ptrace09; kselftest `peeksiginfo.c` pass. |
| S6 | 12 | Tier 7 stretch + hardware watchpoints via DR regs. | GDB `watch *p` triggers on Asterinas. |
| S7 | 13 | proptest model-based fuzzer ("Tracer-Tracee state-machine reference model"). | Run 1 M random transitions vs the Linux oracle in QEMU; zero divergences. |
| S8 | 14–15 | Rust-native `asterdbg` facade + sample debugger. | Demo: attach to a recursive grep, set watchpoint, single-step into a syscall. |
| S9 | 16 | Documentation, design report, video demo. | Final report + PR-ready upstream branch. |

LTP cases mapped to tiers (verified from `ltp/testcases/kernel/syscalls/ptrace`):
- `ptrace01` basic attach → S1; `ptrace02` fork/exec → S5; `ptrace03` mem/regs → S2; `ptrace04` signals → S5; `ptrace05` perms → S1 (uses Yama #3018); `ptrace06` concurrency → S5; `ptrace07` cont/single-step → S3; `ptrace08` syscall trap → S4; `ptrace09` exit → S5; `ptrace10`/`ptrace11` advanced register / regressions → S6.

kselftest cases: `get_syscall_info.c`, `set_syscall_info.c` → S4; `peeksiginfo.c` → S5; `vmaccess.c` → S2; `get_set_sud.c` (syscall-user-dispatch) → explicitly out-of-scope, document as such.

## 4. Hard subset — signal-stop semantics + thread-group tracing

These two are the tar-pit. Budget 40 % of all engineering hours here.

**Signal-stop semantics**: every signal delivered to a traced task must transit `JOBCTL_TRACED`, surface to the tracer via `wait()`, be inspectable / replaceable via PTRACE_GETSIGINFO/SETSIGINFO, and re-injected (or swallowed) via PTRACE_CONT's `data` argument. Asterinas's signal pipeline (#3203, #3120 mapping CPU exceptions to fault signals) is still in flux. Concretely:

- group-stop: SIGSTOP/SIGTSTP/SIGTTIN/SIGTTOU stop *all* threads — must coordinate with thread-group bookkeeping.
- PTRACE_EVENT_STOP (used by SEIZE) — distinct from signal-stop, requires `PTRACE_LISTEN` to resume.
- PTRACE_INTERRUPT vs SIGSTOP precedence rules.
- syscall-enter-stop + group-stop racing: must serialise so the tracer observes a deterministic order.

We will fork the LTP `ptrace04` and write a smaller reproducer set first, then graduate to LTP. Reference: Linux `kernel/signal.c::ptrace_signal()` and `kernel/ptrace.c::ptrace_resume()`.

**Thread-group tracing**: PTRACE_O_TRACECLONE makes every cloned thread auto-attached. Each thread has its own ptrace state but shares mm/sighand. Key pitfalls:

- Reaping of a traced thread — `release_task()` ordering — Asterinas uses Rust Arc/Weak rather than refcounts, so the equivalent of "do_notify_parent" must drop the right reference.
- `wait()`'s `__WALL` / `__WCLONE` semantics across thread groups.
- Stop-all-threads behaviour during PTRACE_INTERRUPT on a thread-group leader.

## 5. Innovation hooks (the 10 % bucket)

1. **Capability-typed tracer handle**: `TracerHandle<'a>` borrows the tracee's `Pid`, prevents two simultaneous attachers at compile time, and forbids reading freed memory by tying `PeekedSlice` lifetimes to `TracerHandle`.
2. **Proptest state-machine vs Linux oracle**: novel; we are unaware of any prior systems contest entry that runs differential testing against Linux kernel as oracle. This will read very well to academic reviewers.
3. **Crash-only debugger**: leveraging Asterinas's Framekernel TCB minimisation, the tracer never gets to touch privileged memory directly; all accesses route through OSTD-validated APIs. Document this as a security argument.
4. **GDB remote-stub bridge**: small in-kernel agent that turns ptrace into a TCP gdbserver, so GDB can be remote. Cheap to build, demo-friendly.

## 6. Risk and mitigation

- *Upstream lands faster than us*: rebase weekly; treat upstream as a teammate, not a competitor. Submit own PRs upstream — this becomes a story for the final report.
- *Signal pipeline churn*: pin to a known-good Asterinas commit for the contest cutoff snapshot; document the divergence.
- *LTP relies on glibc features Asterinas lacks*: maintain a `SKIP_TEST_IF` allowlist (machinery exists via merged PR #3058) and account for it in the report's "100 % implementation" claim.

## 7. Bottom line

proj9 is *more* attractive after this dive than after Stage 1, but **only** if reframed: the team's value-add is "complete + verify + Rust-ify" rather than "implement". Concrete plan above gives a realistic path from upstream-PR-snapshot to a green LTP/kselftest run plus a defensible innovation story, all inside a 16-week window.
