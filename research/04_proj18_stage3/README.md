# proj18 Stage 3 调研索引

> Stage 2 决定选 proj18 后做的 4 个独立子领域并行深挖 + 实施计划合成。

| 文件 | 内容 |
|---|---|
| [`A_corpus_acquisition.md`](A_corpus_acquisition.md) | 历届 CSCC OS 仓库的可获取性、年份分布、版权、首批 10 仓库实测 URL，含 `oscomp/os-competition-info` 主索引 |
| [`B_static_analysis_stack.md`](B_static_analysis_stack.md) | tree-sitter / ra_ap_* / MinHash 参数；重审 CodeQL（Rust 已 GA，仍跳过新理由）/ Joern；7 维抽取 S-expression 草案；完整 pyproject.toml + Cargo.toml |
| [`C_agent_runtime_ui.md`](C_agent_runtime_ui.md) | Anthropic SDK + 自研 200 行 orchestrator；prompt cache 1h TTL 三段断点；citation grounding via `[[ex:N]]`；CI verifier 完整脚本；visx 取代 react-flow；模型分级 Sonnet→Haiku；$7-$77 成本估算 |
| [`D_fingerprint_grading.md`](D_fingerprint_grading.md) | 7 维 fingerprint 逐维抽取 spec（已验证 rCore/xv6/ArceOS/Asterinas/Starry 真实代码）；4 类 fact taxonomy（F-CITE/F-NUM/F-STRUCT/F-CAT）+ 验证机制；20 红队测例；答辩三连发 |
| [`E_implementation_plan.md`](E_implementation_plan.md) | **合成：4 个月修订 sprint 计划 + 第一两周可执行 checklist + 横向 Skill 路线 + 风险登记** |

## 一图流（决策已锁定）

- **数据**：`os.educg.net` 账号 Day 0 拿，`oscomp/os-competition-info` 当主索引，~58 一等奖 + 2 基线
- **抽取**：tree-sitter + ra_ap_*；拒 CodeQL（闭源 + schema 错位）/ Joern（太新）
- **Agent**：Anthropic SDK + 自研 orchestrator，**拒 Agent SDK**（自主 loop 扩 hallucination 面）
- **缓存**：1h TTL（5min 在 60 仓库批跑会失效）三段断点；命中率 78.7-99%
- **模型**：Describe 用 Sonnet 4.5，Compare prose 用 Haiku 4.5
- **citation**：所有 prose 只能引 `[[ex:N]]` 反查 `excerpts.jsonl`，sha256 + CI 硬门
- **结构判定**：LLM 只能从菜单选择，禁止自由式生成
- **前端**：Next.js 15 + visx（非 react-flow）+ Monaco-diff
- **复现**：Docker Compose + corpus.lock.json + cached outputs 入仓
