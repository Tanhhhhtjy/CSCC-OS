# proj18 Stage 3 — 实施计划与首两周可执行 checklist

> 综合 4 份子报告（A 语料 / B 静态分析栈 / C Agent runtime+UI / D 指纹设计+评分操作化）形成的可执行计划。

## 1. 全局技术决策（已锁定）

### 数据层
- **主索引**：`github.com/oscomp/os-competition-info`（GPL-3.0，285★，5 年 winners 单点真相）
- **语料目标**：5 年一等奖 ~58 仓库 + 2 个基线（rCore-Tutorial-v3、xv6-riscv）
- **第一道闸**：注册 1 个 `os.educg.net` 账号（**Day 1 必做**）解锁 2023+ 匿名化命名空间
- **预期分布**：Rust 70% / C 20% / 混合 10%；GPL-3.0 主导，~5-10% 无 LICENSE → 走 "facts-only, no source bytes" 通道
- **重保**：每仓库 clone 时记 commit SHA 到 `corpus.lock.json`，钉死可复现

### 静态分析层
- **语法层**：`tree-sitter-rust 0.24.2` + `tree-sitter-c 0.24.2` + cpp + asm + makefile + python
- **Rust 语义层**：`ra_ap_syntax / ra_ap_hir / ra_ap_ide` 精确钉版（`=0.0.286` 形式，月级 API 变动）
- **拒 CodeQL/Joern**：理由更新——CodeQL Rust 已 GA（2025-10-14），但闭源 extractor 不利可复现 + schema 偏 security/taint；Joern `rust2cpg` 2026-4 才初始 AST，太新
- **相似度**：MinHash num_perm=128, b=16, r=8；每维阈值 0.50-0.85（trap frame 最严，IPC 最松）
- **Embedding**：bge-m3（MIT, 0.6B）v1；Qwen3-Embedding-0.6B 作 drop-in 升级

### Agent 层
- **运行时**：Anthropic Client SDK + ~200 行自研 orchestrator
- **明确拒用**：Claude Agent SDK 的自主 tool loop（扩大 hallucination 面）、LangChain、LlamaIndex
- **缓存布局**（必须 1h TTL，5min 在 60 仓库批跑中会失效）：
  - System prompt: 1h TTL, ~6k tokens
  - 语料统计上下文: 1h TTL, ~4k tokens
  - 单类别 guide: 5m TTL, ~2k tokens
  - 命中率目标：Describe 78.7%、Compare ~99%
- **模型分级**：Describe 用 **Sonnet 4.5**；Compare 散文用 **Haiku 4.5**（3× 更便宜）
- **预算**：$7 乐观 / $77 悲观 / 完整一次跑（60 repos + 1800 pair-compares）

### Citation grounding（架空"精准无误"的承重柱）
- **机制**：所有事实必须引用 `excerpts.jsonl` 中的 `[[ex:N]]` ID；prose 中**禁止**自由 file path 出现
- **抽取**：Extract-Agent（确定性，无 LLM）在分析时写 `excerpts.jsonl`，每条含 `id, file, line_start, line_end, sha256, content`
- **验证**：post-hoc `verify_citations.py` 在 CI 中：对每个 `[[ex:N]]` 反查 excerpt，重算 sha256，content drift = build fail
- **fact 四分类**（来自子报告 D）：
  - **F-CITE**（file:line 引用）—— P0 硬门，错一个 = ship 阻断
  - **F-STRUCT**（结构判定，如 "scheduler = round-robin"）—— P0 硬门，且**只允许菜单选择，禁止自由式生成**
  - **F-NUM**（数值断言）—— P1，自动回写到 extractor 数值
  - **F-CAT**（散文形容词）—— P2，密度上限 15%

### 前端层
- **栈**：Next.js 15 App Router + Tailwind + **visx**（取代 Stage 2 的 react-flow，类别错配——60×60 热图不是图论问题）
- **react-flow 保留**仅用于调用图片段子视图
- **diff 视图**：Monaco-diff
- **注意**：v15 默认 cache 行为变了，需要 `dynamic = 'force-static'`

### 可复现性
- `docker-compose.yml` + `corpus.lock.json`（commit SHA 钉死） + cached LLM outputs（shipped in repo）
- `make replay` 完全离线
- `make replay-from-scratch` 重新调 API

## 2. 修订后的 4 个月 Sprint 计划

> 取代 Stage 2 中的 sprint 草案；并入 4 个 Stage 3 子报告的具体技术决策。

| Sprint | 周次 | 核心交付 | 验收 |
|---|---|---|---|
| **S0 准备** | W0 | os.educg.net 账号、首批 10 仓库 clone 验证、CI 骨架、`corpus.lock.json` v0 | 10/10 仓库可读，CI 跑通 hello-world |
| **S1 语料 + 抽取骨架** | W1-2 | 完整语料 clone（~58 + 2 基线）；tree-sitter + ra_ap 抽取 boot path & trap frame 两维 | 全语料 ≥95% 仓库无错走完抽取；2 维 `fingerprint.json` 出 |
| **S2 抽取扩到 7 维** | W3-4 | scheduler / mm / fs / ipc / syscall 五维上线，全部用 tree-sitter S-expression + AST 规则；F-STRUCT 菜单确定 | rCore/xv6/ArceOS/Asterinas/Starry 5 个基线 7 维全过 |
| **S3 Citation 管线** | W5-6 | `excerpts.jsonl` 写入器；`[[ex:N]]` ID 协议；`verify_citations.py` CI 集成；20 红队测例先存档 | 红队测 20/20 在抽取层正确分类 |
| **S4 Describe-Agent** | W7-8 | Anthropic SDK 直调 + orchestrator（~200 LOC）；prompt cache 三段断点；F-CITE 硬门 | 单仓库 Describe 报告（grounded，含 `[[ex:N]]`），10 红队 0 hallucination |
| **S5 Compare-Agent + MinHash** | W9-10 | Haiku 4.5 跑 pair-compare 散文；MinHash Jaccard 矩阵；阈值标定 | 1800 pair 在 < $25 跑完，矩阵自洽（rCore 同源应聚类） |
| **S6 红队全集 + 评估** | W11-12 | 20 红队全跑（含"注释撒谎说 MLFQ 实为 round-robin"、宏生成 syscall 表、rCore fork 重命名字段等） | 20/20 全过 |
| **S7 仪表盘 + 提交 workflow** | W13-14 | Next.js + visx 热图 + 小多图 + Monaco-diff；"提交新仓库"端到端流程 | 评委开仪表盘 2 分钟内得到新仓库的全报告 |
| **S8 加固 + 复现 + 录屏** | W15-16 | Docker Compose；`make replay` 离线运行；6 分钟演示视频；冻结 `v1.0-cscc` tag | 视频 + 复现镜像 + 离线运行不依赖外网 |

**基础任务（一/二/三）在 S4 闭环，进阶任务（四/五）在 S6 闭环，S7-S8 是开放分（最高 25%）的纯优化空间。**

## 3. 答辩三连发（针对"这不是 RAG 吗"的拦截）

来自子报告 D：

1. **红队 demo #1（注释 vs 代码冲突）**：现场跑一个仓库——其源码里调度器的注释写"MLFQ"，但 AST 分析判定为 round-robin。系统的输出**反对**注释，且每条结构性判定都有 `[[ex:N]]` 反查到具体行号。**关键消息**：我们不相信 LLM，我们相信 AST。
2. **40 仓库 ≤3 min 决定性管线**：在评委面前一次性跑 40 个历届仓库的全流程，时间戳显示绝大部分时间消耗在确定性抽取而非 LLM 调用——**LLM 只是散文外壳**。
3. **实时 citation validator**：仪表盘上一个 "Verify" 按钮，点击后逐条核对所有 `[[ex:N]]`，sha256 + content drift 检查全部通过 = 绿灯。任何一条假引用 = 红灯，直接拒绝展示该报告。

**两分钟答辩稿**：
> "KFC-agent 走的是 hypothesis→trace→refine 的 agent loop——它的价值在 agent 架构。我们走的是另一条路：**比较学本体（comparative ontology）+ 确定性抽取 + LLM 仅作散文表层**。这意味着我们的事实由 AST 决定、不由 token 采样决定，所以可以做到'精准无误'是评分基线而非追求。" + 演示红队 #1。

## 4. 第一两周可执行 checklist

### Day 0（动手前 1 小时）
- [ ] 在 `os.educg.net` 注册账号，保存 cookie 到 `.secrets/educg.cookie`（chmod 600，加 `.gitignore`）
- [ ] Anthropic API key 准备好（已有就过）

### W1
- [ ] `git checkout -b feat/proj18-bootstrap`
- [ ] `mkdir -p proj18/{corpus,fingerprints,reports,excerpts,tools,web}`
- [ ] `proj18/corpus.lock.json` 初始化为空对象 `{}`
- [ ] 写 `proj18/tools/clone_corpus.py`：从 `oscomp/os-competition-info` 读取 winners 列表，clone 到 `corpus/<year>/<team>/`，写入 `corpus.lock.json[<repo>] = {url, sha, year, track, prize, license}`
- [ ] 跑首批 10 仓库（A 报告 §8 已给清单），核验全部可 clone
- [ ] 写 `tools/verify_citations.py` 骨架（C 报告已给完整脚本，照抄）
- [ ] 写 `Dockerfile` + `docker-compose.yml`：python 3.12 + cargo + ra_ap + tree-sitter
- [ ] CI（GitHub Actions）：lint + `verify_citations.py --dry-run` + `pytest tools/`
- [ ] 起草 RFC：`proj18/RFC-001-fingerprint-schema.md`，定义 `fingerprint.json` v0 schema（D 报告 Part 1 直接转写）

### W2
- [ ] 实现 boot-path 维度抽取（D 报告里已经验证过 rCore `_start` 与 xv6 `_entry` 的真实代码）
- [ ] 实现 trap-frame 维度抽取（rCore TrapContext 33 字段、xv6 trapframe 35 字段，已验证）
- [ ] 跑通基线两仓库（rCore-Tutorial-v3 + xv6-riscv）的 2 维 fingerprint，输出 `fingerprints/<repo>.json`
- [ ] 写 20 红队测例的 README + 占位仓库（D 报告 Part 2 已列全）
- [ ] 启动一个新 skill：`fingerprint-extractor-template`（给后续 5 维抽取作脚手架）
- [ ] **S1 验收**：在 5 个基线仓库（rCore-Tutorial-v3 / xv6-riscv / ArceOS / Asterinas / Starry）的 boot + trap-frame 两维通过；fingerprint JSON schema 写死

## 5. 需要先落地的横向 Skill（按优先级）

1. **`fingerprint-extractor-template`**（W2 起手）—— 7 个抽取器的共同骨架：tree-sitter 加载 + S-expression 查询 + AST 规则 + 输出 JSON。其余 5 维抽取直接派生。
2. **`citation-grounded-prose`**（W7 起手）—— Describe-Agent 的 prompt 模板 + `[[ex:N]]` 协议 + verifier hook。
3. **`menu-only-structural-verdict`**（W3-4）—— 强制 LLM 只能从 `{round-robin, MLFQ, CFS-like, BFS, custom}` 这种封闭集合输出，禁止自由式描述。constrained decoding 或 post-hoc lint。
4. **`os-comparative-ontology`**（W3 起，活文档）—— 七维 fingerprint 跨 rCore/xv6/ArceOS/Asterinas/Starry 的等价规则。

## 6. 风险登记（按优先）

| ID | 风险 | 触发指标 | 缓解 |
|---|---|---|---|
| R1 | 2023+ 仓库 ACL 锁 | clone 失败率 > 10% | os.educg.net 账号 Day 0 拿到；不行就发邮件给 oscomp 维护者请求 read-only mirror |
| R2 | "Just RAG" 评委质疑 | 答辩时 < 30s 内被打断 | 红队 demo #1 + 三连发已备；现场 1 个仓库实跑 |
| R3 | 一条假 citation 在 ship 时漏过 | verify_citations.py 漏报 | F-CITE = P0 硬门，sha256 校验全文 + CI 阻塞 |
| R4 | 七维某维分类系统失败（如 IPC 难分） | 红队 ≥3 个连错 | 该维降级为 "未分类 / 不确定"，不强行猜；散文报告里诚实标注 "本维 IPC 无法确定" |
| R5 | 仪表盘吃掉太多时间 | S7 末没做完 | S4-S6 已闭环必做项；S7-S8 是优化分；仪表盘可以最后一周才动 |
| R6 | 成本超支 | 单次 run > $100 | Haiku 接管 Compare（已默认）；缓存命中率监控；批量改增量 |
| R7 | 团队"no-LLM-in-the-structural-loop"纪律滑坡 | code review 发现结构判定走 prompt | CI lint：所有 F-STRUCT 必须经 menu_select() 通过；自由 prompt 调用结构判定 = 阻断 |

## 7. 下一步

确认本计划后我可以：
1. 直接照 Day 0 + W1 checklist 起手（建分支、写 clone_corpus.py、写 verify_citations.py 骨架、Dockerfile 等）
2. 把 4 份子报告的精华摘到 `research/README.md` 索引
3. 把这份 Stage 3 计划 commit + push 到 `research/topic-selection` 分支
