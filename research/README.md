# 2026 CSCC 操作系统功能挑战赛道 — 选题调研

> 全部 63 道题目的系统化调研，最终建议主攻 **proj61（Agent-OS 内核）**，保底 **proj47（Agent 记忆向量检索 I/O）**。

## 文件索引

| 文件 | 内容 |
|---|---|
| [`00_stage1_wide_filter.md`](00_stage1_wide_filter.md) | Stage 1 宽筛：63 题按主题分 5 组并行打分（V/R/H/S/C 五维），跨组 Top 12 + 黑名单 + 五个战略选型框 |
| [`01_stage2_final_recommendation.md`](01_stage2_final_recommendation.md) | Stage 2 最终决策：Top 10 重排、主攻 + 保底 + 弃选、前两周启动 checklist、横向 Skill 路线图 |
| `02_deep_dives/proj*.md` | 13 份逐题深度可行性报告（~1700-2700 词/份），含 proj6/9/11/18/22/32/34/41/47/49/52/53/61 |
| [`03_final_comparison.md`](03_final_comparison.md) | 最终 5 题（proj18/41/52/53/61）深度对比表，含 AI 在每题对应技术栈的子系统级成熟度评分 |
| [`04_proj18_stage3/`](04_proj18_stage3/) | **选定 proj18 后的 Stage 3 深挖**：语料获取 / 静态分析栈 / Agent runtime+UI / 指纹+评分操作化 / 4 个月实施计划 + 首两周 checklist |

## 一句话结论

**主攻 proj61 — 面向 AI 智能体的操作系统内核（Agent-OS）**。Stage 2 评分 9/10。

理由：Claude Code 团队用 Claude Code 开发"为 Claude Code 这样的 agent 服务"的 OS——dogfooding 闭环最完美；纯 RISC-V QEMU 零硬件；无上游合入风险（区别于 proj9 已被 Asterinas PR #2984 抢先）；无往届优胜拥堵（区别于 proj34 xv6 VFS 已有两届优胜 + 老师参考实现）；8 sprint 路径已具体到具名数据结构。

**保底 proj47 — Agent 记忆向量检索 I/O**。Stage 2 评分 8.5/10。DiskANN 现已 99.9% Rust 化，选 Vamana（节点+边正好 4 KiB），io_uring `URING_CMD` NVMe passthrough，目标 SIFT1M recall@10 ≥ 0.95、单线程 5k QPS、16 线程 30k QPS、p99 ≤ 3 ms。

## Stage 2 重要修正（区别于 Stage 1 表象分）

| 题 | Stage 1 | Stage 2 | 修正原因 |
|---|---|---|---|
| proj9（Asterinas ptrace） | 23/25 | 7/10 | Asterinas 上游 PR #2984 已于 2026-04-26 合入，"从零实现" 叙事破产 |
| proj34（xv6 VFS+HAL） | 23/25 | 7/10 | 两个往届优胜（静春山、RuOK 2025）+ 老师测试仓库已发布，差异化窗口窄 |
| proj11（Asterinas dm-crypt） | 23/25 | 6/10 | Asterinas `kernel/comps/` 无 device-mapper 抽象层，需先造 `aster-dm`，工作量翻倍 |

## 黑名单（不建议任何 vibe-coding 团队选择）

**硬件锁死**：proj4、proj7、proj30、proj31、proj36、proj39、proj57
**评测模糊 / 研究开放问题**：proj13、proj42、proj45
**用户态生态缺失**：proj62（virtio-gpu / Vulkan 整套要从头）
**配套较弱**：proj17、proj43

## 调研方法

1. **Stage 1**（宽筛）：5 个 subagent 并行，每个负责 12–19 道题的同主题群。打分维度 V/R/H/S/C 各 1–5 分。每个 subagent 输出独立的群内排行 + 战略观察。
2. **Stage 2**（深挖）：根据 Stage 1 跨组 Top 12 + 团队偏好（不限硬件 / 假设技能全），创建 Agent Team `cscc-os-stage2`，5 个 teammate 各深挖 2 题，输出 ~1700 词/题的可行性报告（技术路径 / 4 个月 sprint 计划 / Skill 设计 / Demo & 评测 / 风险 & 缓解 / 1–10 终评分）。
3. **合成**：team-lead 根据 10 份深挖报告 + 跨题战略洞察，写出本目录的最终决策。

## 接下来该做的第一件事

打开 `01_stage2_final_recommendation.md`，照着"前两周启动 checklist"动手：
- Fork rCore-Tutorial-v3，锁定 commit
- 写 demo scenario 文档（**锁死防 scope creep**）
- 起草 `agent-call` ABI v0.1（TLV，严禁内核 `serde_json`）
- 起草 `agent-runtime-spec` skill 骨架

> 任何 OS 接口变更走 RFC.md → codex review → 合并代码 三步走。优先把 `kernel-rfc-template` 和 `tdd-syscall-loop` 两个横向 skill 落地。
