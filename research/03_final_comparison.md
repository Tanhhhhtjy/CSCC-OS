# 五题最终深度对比表 — proj18 · proj41 · proj52 · proj53 · proj61

> 基于 5 份 ~1700-2700 词深度报告（`02_deep_dives/projXX.md`）综合而成。

## 一句话定位

| code | 一句话 | Stage 2 评分 |
|---|---|---|
| **proj61** | 给 rCore-Tutorial-v3 加 Agent-PCB + ACR + MCP-shape TLV ABI + Agent Loop 调度 | **9.0 / 10** |
| **proj18** | tree-sitter + rust-analyzer + LLM 把历届 CSCC OS 仓库做"七维结构指纹"+ 谱系/创新分析 | **7.5 / 10** |
| **proj41** | WasmEdge + WASI-NN 把 3 家国产 NPU（Sophon / RKNN / Sunrise）做统一推理 plugin | **6.5 / 10** |
| **proj53** | 把 TAIC 软件模型 port 到 ArceOS（asyncos 家族贡献） | **6.5 / 10** |
| **proj52** | xv6-riscv + 内核态 int8 perceptron 调度器（RVV 选用，"智能 OS"） | **6.0 / 10** |

## 主对比表

| 维度 | proj61 | proj18 | proj41 | proj52 | proj53 |
|---|---|---|---|---|---|
| **基础语言/栈** | Rust no_std + RISC-V QEMU | TypeScript + Rust(tree-sitter) + Claude API | C/C++ + Rust + WasmEdge | **C 语言强制** + RV64 + RVV1.0 | Rust async no_std + RISC-V |
| **基础内核/项目** | rCore-Tutorial-v3 main | 自起平台（Next.js + Neo4j 等） | WasmEdge master | xv6-riscv | ArceOS（推荐）/ rCore-N / ReL4 |
| **硬件需求** | 零（QEMU） | 零 | **3 块 NPU 板**（RK3588 + Sophon + Sunrise，~¥4-8k） | 零（QEMU/Gem5）；选用 VisionFive2 | 零（taic-qemu fork） |
| **上游已抢/拥堵** | 无 | 有 5+ 同主题往届，但无人做"OS 结构指纹" | **WasmEdge 已有 11 个 backend，无 Sophon/RKNN/Sunrise**（issue #3500 unowned）| 无（但 Stage 1 C=1 评测模糊） | **极高**：导师同门 PhD 论文家族占位（赵方亮 TAIC、廖东海/李龙昊 ReL4） |
| **评测可观察性** | QEMU 内核日志 + ratatui 仪表盘 + 单元测试 | 历届报告检索准确率 + judging 仓库 ground-truth | YOLOv5n@COCO val2017 mAP/FPS/p99，3 backend 跨平台一致性 | 混合负载吞吐 + 模型替换 A/B（"换坏模型变更差"= 真智能） | echo server 吞吐 + 中断转发 jitter，A/B sync 对照 |
| **可一键 demo** | ✅ 6 分钟 QEMU 录屏 | △ 需 Web UI + 后端 + 数据 | △ 需 3 块板同台展示 | ✅ 录屏即可 | ✅ taic-qemu 录屏 |
| **Stage 1 → Stage 2 修正** | 25 → 9（持平） | 23 → 7.5（条件通过） | 12 → 6.5（**显著上调**，因 WasmEdge 上游确有空白） | 16 → 6.0（持平，**RVV-in-kernel 是叙事而非必需**） | 20 → 6.5（**显著下调**，incumbency 风险被低估） |

## Claude Code / AI 在各题"对应技术栈"的成熟度（按子系统 1-5 评）

> 这是你专门让我看 proj52/53 的内容，附 proj61/41 同维度作横向对照。

### proj52 — RISC-V + RVV + C 内核

| 子系统 | AI 熟练度 | 说明 |
|---|---|---|
| RV64 boot / trap / Privileged ISA (M/S/U, satp, mstatus, mcause, Sv39 PT, PMP) | **4** | xv6-riscv 是训练数据黄金集，AI 写得很扎实 |
| OpenSBI 扩展 / vendor extension slot | **3.5** | sbi_ecall_extension_t 结构 AI 知道，但具体 vendor 扩展号约定要查文档 |
| **kernel-mode RVV 1.0 intrinsics** | **2.5 ⚠️** | **AI 训练数据被 pre-1.0 RVV 0.7.1（THead 玄铁）污染**；Linux 6.8 `kernel_vector_begin/end` 模式太新，AI 经常混搭 0.7.1 与 1.0 语法 |
| 内核态轻量 ML 推理（int8 perceptron / 小型决策树） | **4** | 算法本身简单，AI 写得快 |
| 评测/基准/profile pipeline | **4.5** | 标准技能 |

**结论**：proj52 的硬骨头（kernel-mode RVV save/restore + `vec_frame` 懒保存）正好压在 AI 最弱的格子。建议如选 proj52，**直接放弃 RVV，把 RVV 降级成 "我们做了 RVV 兼容性测试，没用作主要推理路径" 的叙事**（agent 报告里也建议这么 descope）。

### proj53 — 异步 Rust 内核

| 子系统 | AI 熟练度 | 说明 |
|---|---|---|
| Rust async/await 用户态 idiom（Pin/Unpin/Waker/Future） | **4.5** | tokio/embassy 训练充足 |
| no_std 自定义 executor（Pin 投影、`!Send` future、原子 waker） | **3.5** | 能写但要紧密 review |
| **interrupt + async 健全性（无 missed-wakeup race）** | **2.5 ⚠️** | **AI 经常写出 missed-wakeup 死锁的代码**（在 enqueue 与 set_ready 之间没有 fence 或者顺序错） |
| 能力型微内核（seL4 cap-transfer 语义） | **2.5 ⚠️** | AI 对 cap 模型只有表层理解，IPC fast path 经常写错不变量 |
| RISC-V Rust 内核上下文（rCore-N / ArceOS / Asterinas / ReL4） | **3.0** | rCore 训练较好，其余项目训练稀薄 |

**结论**：proj53 的核心难点（**async + 中断 的健全性**）正好是 AI 最弱的格子。即使选 path (d)（TAIC port to ArceOS）这一相对软的路径，仍要靠人工写关键临界区。

### proj61 — Agent-OS（参考线）

| 子系统 | AI 熟练度 |
|---|---|
| Rust 系统编程 + rCore 风格内核 | **4.5** |
| Syscall 加新表项 / 用户态 libc wrapper | **4.5** |
| Lock-free ring buffer（ACR） | **3.5**（需 miri + 人审） |
| easy-fs 扩展（属性 sidecar） | **4** |
| 调度器扩展（AgentBlocked 状态 + event bus） | **4** |
| MCP 二进制 ABI 设计 | **4.5** |

**结论**：proj61 没有任何子系统命中 AI 弱项。

### proj18 — OS 结构指纹分析

| 子系统 | AI 熟练度 |
|---|---|
| tree-sitter 解析器 / 查询规则 | **4** |
| rust-analyzer LSP 集成 | **3.5** |
| 向量检索 / RAG | **5** |
| Next.js / 仪表盘 | **5** |
| OS 结构七维抽取（boot/trap/sched/mm/fs/ipc/syscall） | **4**（规则化抽取的部分） |
| LLM 散文生成（要严防幻觉） | **3.5**（必须人审 + 引用归一化） |

### proj41 — WasmEdge + WASI-NN

| 子系统 | AI 熟练度 |
|---|---|
| WasmEdge plugin C++ API | **4** |
| WASI-NN spec 扩展（graph_encoding） | **4** |
| Sophon LIBSOPHON / SAIL SDK | **4**（开放、中英文档全） |
| 瑞芯微 RKNN-Toolkit2 / rknn_api | **4**（同上） |
| **地平线 Sunrise BPU SDK（libdnn.so）** | **2 ⚠️** | **闭源 .so + 文档薄 + 开发者门户登录**，AI 帮不上 |
| COCO/YOLO benchmark pipeline | **5** |

## 战略对比

| 维度 | proj61 | proj18 | proj41 | proj52 | proj53 |
|---|---|---|---|---|---|
| 上限（"非对称大赢"潜力） | 高 | 高 | 中 | 中 | 中 |
| 下限（保底完赛概率） | 高 | 中（差异化失败可能变"RAG over repos"） | 低（板子断供/SDK 改 ABI） | 中（RVV 难度若没 descope 会拖 sprint） | 中（async 健全性 bug 难定位） |
| 选题独特性 | **强**（无人做） | 中 | **强**（WasmEdge 上游无此 3 backend） | 弱（智能 OS 框架很多） | 弱（incumbent 占位） |
| 评委可见性 | 高（demo 直观） | 中（要看报告/UI） | 中（要看 3 板一致性） | 低（"intelligent" 抽象） | 中（性能 A/B 易懂） |
| 4 个月 ddl 风险 | 低 | 中 | **高**（硬件 + 多 SDK） | 中 | 中-高 |
| Vibe coding 杠杆率 | 极高 | 极高 | 中（vendor SDK 部分非 AI 化） | 中（C + RVV 限制） | 中-低（关键临界区 AI 不可替代） |
| 与团队 Claude Code 招牌叙事的契合 | **完美**（dogfooding） | **很好**（agent-on-codebase 即是产品） | 一般 | 一般 | 一般 |
| 失败可切换性 | proj47 是天然 fallback | 切到 proj61 需要换技术栈 | 切到 1 块板的 demo 可以保命 | 可切到非 ML 的"自适应"OS | 可切到非 TAIC 的纯 async 调度器 |

## 决策建议（按确定性排序）

1. **🥇 主攻 proj61（9.0/10）**——综合最强，没有任何一项指标落入红色。下限最高、上限不亚于其它任一。
2. **🥈 备胎 proj18（7.5/10）**——若团队人力 ≥ 4 且其中 ≥ 1 人愿意死守 "结构由确定性管线提取、LLM 只写散文" 的纪律，可与 proj61 并行（不同人）；否则不要并行。
3. **❌ proj41（6.5）**：唯一进入第一梯队的"硬件题"，但 3 NPU 板 + 闭源 Sunrise SDK 是不可消除的死区。除非团队**已经拥有 RK3588 + Sophon 两块板**，否则不推荐。
4. **❌ proj53（6.5）**：incumbency 风险 + AI 在中断/async 健全性的弱点 = 双杀。若团队中**没有**已经做过 Rust 内核异步的人，不要碰。
5. **❌ proj52（6.0）**：可做但卖点（RVV + 内核 ML）正好踩 AI 弱区。题目本身的 C=1（demo-only 风险）也没有被 Stage 2 充分解开。

## 我个人的建议

**主攻 proj61，不要并行**。proj18 即使做也是把它当 proj61 的姊妹项目——共用结构抽取技术、共用历届仓库语料，可以给 proj61 的"哪些 syscall 设计已经被前人验证过"提供旁证。proj41/52/53 三题如果你的团队有特别强的兴趣点（比如有人就是想啃 RVV、有人就是想啃 async + interrupt），那它们是合理的"个人爱好升级版"选择，但不是最优竞赛策略。
