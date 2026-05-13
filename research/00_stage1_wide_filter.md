# CSCC 2026 OS-Functional 选题宽筛报告（Stage 1）

5 个并行调研 agent，共覆盖 63 题。打分轴：V 适配 vibe-coding · R 资料丰富度 · H 硬件友好度 · S Skill 复用空间 · C 4 个月可完成性。满分 25。

## 跨组 Top 12（按总分 + 战略价值）

| 排名 | code | 题目 | 总分 | 类型 | 一句话定位 |
|---|---|---|---|---|---|
| 1 | proj61 | Agent-OS 内核（rCore/uCore + RISC-V QEMU） | **25** | 内核+Agent | Claude Code 团队的"自己造给自己用的 OS"——dogfooding 满分 |
| 2 | proj32 | 基于 eBPF 的多 Agent 异常监测 | **24** | eBPF+AI | 把 LLM 语义意图与 syscall 行为做关联，可用 Claude Code 当被监控 agent |
| 3 | proj6 | eBPF 网络服务加速 + 双端缓存 | **24** | eBPF+网络 | 最成熟的系统软件方向，参考代码海量 |
| 4 | proj18 | 小型 OS 分析比对智能体 | **23** | 纯 AI Agent | 一个 Claude-Code-shaped 项目穿着比赛外衣；非对称优势最大 |
| 5 | proj47 | 面向 Agent 记忆的向量检索 I/O 优化 | **23** | 存储+AI | DiskANN/HNSW + I/O 调度，分解清晰，SIFT 标准基准 |
| 6 | proj9 | Asterinas ptrace / 进程调试 | **23** | Rust OS | manpage 驱动、~40 个 PTRACE_* 可逐个 TDD |
| 7 | proj11 | Asterinas dm-crypt + dm-verity | **23** | Rust OS+密码 | 算法清晰、RustCrypto 成熟、cryptsetup 当 oracle |
| 8 | proj34 | xv6 VFS + HAL（RISC-V + LoongArch） | **23** | 教学型内核 | 最稳的工程题，纯 QEMU，两个往届优胜参考 |
| 9 | proj54 | OS 课程实验 / 竞赛套件（元任务） | **23** | 教育平台 | AI 擅长批量生成教学素材 |
| 10 | proj22 | OpenHarmony DSoftBus LLM 模糊测试 | **22** | 安全+AI | 组委会提供云容器，纯软件设计 |
| 11 | proj49 | 异构多机器人 PDDL 规划调度 | **22** | 规划+算法 | 官方 state-checker 直接当评测，PDDL 是 LLM 强项 |
| 12 | proj56 | 云原生 + AI 协同的 OS 在线学习平台 | **22** | 全栈+教育 | frontend-design + 后端 + LLM 导师，全栈 vibe-coding |

补充值得提一句但分稍低：proj19（崩溃→上游 patch RAG，22）、proj1（DDS 调优，21）、proj28（openvela FS，20）、proj23（CVE 热补丁 agent，20）、proj53（asyncos，20）。

## 明确避免（这些方向对 Vibe coding 是陷阱）
- **硬件锁死类**：proj4、proj7、proj30、proj31、proj36、proj39、proj57（必须自购 / 借不到的国产专用板，闭源 SDK，AI 帮不上忙）
- **研究开放问题类**：proj13、proj42、proj43、proj45（评测标准模糊、无公开 baseline）
- **graphics/虚拟化大坑**：proj62（用户态 Mesa/Vulkan 整套要从头搭）

## 五个战略选型框（Stage 2 候选）

每个框对应一种参赛策略——你需要选一个或几个进入 Stage 2 深度调研。

### 框 A · "AI 原生 / dogfooding"——proj61 + proj18 + proj32
**叙事**：把 Claude Code 团队的看家本事直接当作论文/答辩主线。proj61 是把 OS 改造成"为 Claude Code 这样的 agent 服务"的内核；proj18 是用 agent 去分析所有历史选手的 OS 代码；proj32 是用 eBPF 监控 agent 的行为。三题共享"AI/Agent + 系统"叙事。

### 框 B · "纯系统硬功夫"——proj6 + proj47 + proj9/11
**叙事**：放弃 AI 噱头，做扎实工程题，比代码质量与性能数字。eBPF 网络加速 / 向量检索 I/O / Rust OS 子系统都有清晰 benchmark，赢家由数字说话。

### 框 C · "教学/平台"——proj54 + proj56 + proj20
**叙事**：发挥 frontend-design + paper-orchestra 的全栈优势，做一个能复用的教学平台。颜值高、demo 易出，但要警惕被评委归类为"包装好的网课"。

### 框 D · "安全 + Agent"——proj22 + proj32 + proj23
**叙事**：信息安全是 2026 年的热点。DSoftBus 模糊测试、eBPF 异常监测、CVE 自动热补丁，三题都可以用 Agent 工作流组装，且都有 skill-able 的结构（可写一个 fuzzer-scaffold、ebpf-template、cve-livepatch skill）。

### 框 E · "稳健工程"——proj34 + proj28 + proj1
**叙事**：风险最低、最容易完赛、demo 清晰。但同时也是参赛人数最多、最难差异化的方向。

## 下一步建议

进入 Stage 2 之前需要你拍板：

1. **战略偏好**：你要"非对称大赢/可能翻车"（框 A、B 中的 proj61/proj18/proj32），还是"稳健完赛"（框 E）？
2. **Stage 2 深挖几个？** 我建议 3 个，正式 `TeamCreate` 一个 4 人小组（1 个 team-lead 负责横向比对 + 3 个深挖专家，每人一题）。
3. **可访问的硬件资源？** 如果有 Loongson / 飞腾 / RPi / Jetson / 国产 NPU 板，部分中段题（38、29、58）的可行性会上调；如果纯 x86+QEMU，那就锁定纯软件题。

我个人优先推荐进入 Stage 2 深挖：**proj61、proj32、proj47**（这是"AI 团队最被低估的纯软件题"组合）。如果你想要更稳的，加 **proj34** 当保底。
