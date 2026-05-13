# CSCC 2026 OS Functional Track — 选题最终报告（Stage 2）

调研流程：5 个并行 subagent 宽筛 63 题（Stage 1） → 5 人 Agent Team 深挖 Top 10（Stage 2） → 本报告合成。10 份详细技术报告在 `stage2/projXX.md`。

## 最终排行

按"深度调研后"重排，分数 = Stage 2 teammate 评分（已修正 Stage 1 表象分）：

| 排名 | code | 题目 | Stage2 | 关键转折 |
|---|---|---|---|---|
| 1 | **proj61** | Agent-OS 内核（rCore + RISC-V QEMU） | **9/10** | 高底 + 高顶；TLV 二进制 ABI、MCP 形状、AgentTask PCB 扩展，8 个 sprint 全闭环 |
| 2 | **proj32** | eBPF 多 Agent 异常监测 | **8.5/10** | 用 MCP JSON-RPC request-id 做 L7↔kernel join key，TPR≥90/FPR<2/overhead≤5%，新颖性已验证 |
| 3 | **proj47** | Agent 记忆向量检索 I/O 优化 | **8.5/10** | DiskANN 已 99.9% Rust 化，选 Vamana（节点+边正好 4KiB），io_uring NVMe passthrough |
| 4 | **proj49** | 异构多机器人 PDDL | **8/10** | 双层求解（UP+FD ⇒ CBS-TA），libMultiRobotPlanning 公开数据集 hedge |
| 5 | **proj22** | OpenHarmony DSoftBus LLM fuzzer | **7.5/10** | 5 个差异化点（状态图、kcov 融合、分布式语料），但要靠落 0-day 上限才高 |
| 6 | **proj18** | 小型 OS 分析比对智能体 | **7.5/10** | 条件通过：必须坚守"结构由确定性管线提取，LLM 只写散文"纪律 |
| 7 | **proj6** | eBPF 网络服务加速 + 双端缓存 | **7.5/10** | 协议组合应选 DNS+Redis（RESP 可在 verifier 内解析；gRPC HPACK+TLS 不行） |
| 8 | **proj34** | xv6 VFS + HAL | **7/10** | 两个往届优胜 + 老师测试仓库已发布，**赛道已挤**；需要 ≤4500 LoC HAL + 崩溃恢复 + 第 4 个 FS 才差异化 |
| 9 | **proj9** | Asterinas ptrace | **7/10** | 上游 PR #2984 已于 2026-04-26 合入，纯实现叙事破产；需重定位为"长尾收尾 + Rust-native asterdbg" |
| 10 | **proj11** | Asterinas dm-crypt + dm-verity | **6/10** | Asterinas 无 device-mapper 抽象，需先造一个 `aster-dm`，工作量翻倍；Plan B 牺牲通用性 |

## 单一推荐：**proj61 — 面向 AI 智能体的操作系统内核（Agent-OS）**

### 为什么是它

1. **Dogfooding 闭环**：你们用 Claude Code 开发"为 Claude Code 这样的 agent 服务"的 OS——团队优势直接变成产品叙事，评委一眼看得见。
2. **底高**：8 个 sprint 全部 map 到具名数据结构（AgentTask PCB / Agent Context Region TLV ring / MCP-shape 二进制 ABI / 属性 sidecar / AgentBlocked 任务状态 + event bus）；必做在 Sprint 4 闭环、选做在 Sprint 6 闭环，时间富余。
3. **顶高**：题目本身就是 open design space，主队伍稀缺，**没有 proj9 那种"上游已经合入"或 proj34 那种"两个优胜 + 老师参考实现"的拥堵**。
4. **纯软件**：RISC-V QEMU，零硬件，纯 Rust。Claude Code + codex review 双链协同顺滑。
5. **可写 skill**：`agent-runtime-spec` + `kernel-syscall-table-extender` 两个新 skill 是高复用资产。

### 前两周启动 checklist

```
W1
□ 锁定 rCore-Tutorial-v3 main 分支 commit，fork 到 research/topic-selection
□ 写 demo 脚本（locked-in scenario，防 scope creep）:
   "一个 toy agent 通过 agent-call syscall 调用三个 'tool'（read_file/list_dir/exec），
    内核调度器给 agent task 优先级，事件总线把 tool 输出推回 agent context region"
□ 起草 agent-call ABI v0.1（TLV、严禁内核 serde_json）→ 写成 RFC.md
□ 起草 agent-runtime-spec skill 骨架（frontmatter + 6 段 checklist）
W2
□ AgentTask = TaskControlBlock 扩展（添加 agent_ctx_region: Option<VirtAddr>）
□ sys_agent_call syscall stub + 第一个返回 ENOSYS 的集成测试通过
□ Agent Context Region 用户态映射 demo（mmap + 自定义 fault handler）
□ 团队分工：A 内核态 ABI/调度、B agent 用户库、C 测试+demo+skill、D 文档+报告
□ 通过 codex review 验证 RFC.md（飞书卡片归档）
```

## 保底（fallback）：**proj47**

若中途发现 proj61 演示效果不达预期，可切换到 proj47——技术路径已被 storage-expert 拆得很细（Vamana + io_uring URING_CMD + TinyLFU + SPFresh-style LSM 写路径），SIFT1M 数字目标明确（recall@10≥0.95、5k QPS / 30k QPS / p99≤3ms），无任何政治风险。

## 明确放弃

- **proj9（ptrace）**：上游已合，叙事破产
- **proj34（xv6 VFS）**：赛道已挤，差异化窗口窄
- **proj11（dm-crypt）**：Asterinas 缺 dm 框架，路径风险大
- 之前 Stage 1 黑名单不变（proj4/7/30/31/36/39/57/13/42/43/45/62）

## 团队跨题应当先写的 Skill（横向资产）

无论选哪题，都建议先把这两个 skill 落地（4 月初一周内）：

1. **`kernel-rfc-template`**：所有内核接口变更先走 RFC.md（动机/ABI/兼容性/测试矩阵/反例），再 codex review，最后合并代码。统一团队所有"想给内核加点东西"的入口。
2. **`tdd-syscall-loop`**：syscall 加 / 改 → 写 LTP / kselftest / 自研测例 → QEMU run → log diff → 红绿循环。把 systematic-debugging + test-driven-development 两个现成 skill 合并成内核版的窄接口。

选 proj61 还应早写 **`agent-runtime-spec`**；选 proj32 还应早写 **`ebpf-program-scaffold` + `mcp-tap-protocol`**。

## 一句话结论

**主攻 proj61；4 月底如果 demo 不顺再切 proj47；其余 8 题不再考虑。**
