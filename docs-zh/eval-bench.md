# 针对您的 gbrain 更改运行真实世界的评估基准

受众：gbrain 维护者和贡献者。如果您正在接触检索（搜索、排名、嵌入、意图分类、查询扩展、源提升、混合融合），这就是您需要的文档。

有关 gbrain-evals 使用的 **NDJSON 线格式**，请参见 [`eval-capture.md`](./eval-capture.md)。本文档是建立在该格式之上的人类开发循环。

## 前提条件：开启贡献者模式

捕获在生产用户中**默认关闭**（隐私优先 —— 无意外数据累积）。贡献者用一行开启：

```bash
# 在 ~/.zshrc 或 ~/.bashrc 中：
export GBRAIN_CONTRIBUTOR_MODE=1
```

验证：

```bash
gbrain query "anything" >/dev/null
psql $DATABASE_URL -c 'SELECT count(*) FROM eval_candidates'   # 应该 > 0
```

要覆盖（无论环境变量如何强制开启/关闭），编辑 `~/.gbrain/config.json`：

```json
{"eval": {"capture": true}}    // 强制开启
{"eval": {"capture": false}}   // 强制关闭
```

显式配置在两个方向上都优于环境变量。

## 4 命令循环

```bash
# ① 捕获：当设置 CONTRIBUTOR_MODE 时写入 eval_candidates。
#   检查收集了什么：
gbrain doctor                                     # 显示捕获失败
psql $DATABASE_URL -c 'SELECT count(*) FROM eval_candidates'

# ② 快照：在代码更改之前冻结基线。
gbrain eval export --since 7d > baseline.ndjson

# ③ 代码更改：做任何您想做的事情 —— 调整 RRF_K、交换嵌入模型、编辑
#    hybrid.ts、添加新的提升源、更改意图分类器。

# ④ 重放：针对当前构建重新运行每个捕获的查询。
gbrain eval replay --against baseline.ndjson
```

输出：

```
Replaying 247 captured queries…
  ...25/247
  ...50/247
  ...
Replayed 247 of 247 captured queries (0 skipped, 0 errored)
Mean Jaccard@k:    0.927
Top-1 stability:   91.5%
Mean latency Δ:    +14ms (current vs captured)

Top 5 regression(s):
  jaccard=0.20  captured=12  current=3   "find every reference to widget-co"
  jaccard=0.43  captured=14  current=8   "show me everything tagged for review"
  jaccard=0.50  captured=8   current=4   "what did alice say about the spec"
  ...
```

三个数字告诉您更改是否可以安全落地：

| 指标 | 含义 | 健康范围 |
|---|---|---|
| **Mean Jaccard@k** | 捕获的检索 slug 与当前运行的 slug 之间的平均重叠。1.0 = 相同集合。 | "中性"更改 ≥0.85。<0.7 表示重大检索变化。 |
| **Top-1 stability** | #1 结果未改变的查询比例。 | 调优通过 ≥85%。<70% 表示顶部漏斗破裂。 |
| **Mean latency Δ** | 当前减去捕获。正数 = 现在更慢。 | 在捕获的 ±50ms 内。任何地方 >2× = 回归警报。 |

## 它实际做什么

`gbrain eval replay` 读取您的 NDJSON 快照，并对每一行：

1. 使用捕获的 `detail` 和 `expand_enabled` 值重新执行相同的操作（`tool_name='search'` 使用 `searchKeyword`，`tool_name='query'` 使用 `hybridSearch`）。
2. 捕获当前的 `retrieved_slugs`（去重，按结果顺序）。
3. 计算捕获的和当前的 slug 集之间的 set-Jaccard。
4. 记录 top-1 匹配（#1 结果是否是相同的 slug？）。
5. 记录延迟增量与捕获的 `latency_ms`。

它**不**计算 MRR 或 nDCG —— 这些需要地面实况相关性标签，而不是基线比较。对于针对真相的指标评估，使用 `gbrain eval --qrels <path>`（遗留 IR-eval 路径，仍支持）。重放工具回答不同的问题："我的代码更改是否移动了检索，以及它移动了哪些查询最多？"

对于第三个评估轴 —— 公共基准、地面实况标签、完整问答管道（不仅仅是检索）—— `gbrain eval longmemeval <dataset.jsonl>`（v0.28.8）针对 gbrain 的混合检索运行 LongMemEval 基准。每个问题都获得一个干净的内存中 PGLite，导入其干草堆，提出问题，假设作为 JSONL 发出 —— 正是 LongMemEval 的 `evaluate_qa.py` 消费的形状。您的 `~/.gbrain` 大脑永远不会被打开。参见下面的 `## Public benchmarks: LongMemEval`。

## 设计上的尽力而为

重放不是纯粹的。捕获和重放之间有三件事可能漂移：

1. **大脑状态** —— 您的大脑现在可能比快照时有更多页面。除非您明确设置固定的语料库，否则平均 Jaccard 会因为新页面符合条件而下降。
2. **嵌入源** —— 如果您在捕获和重放之间更改了 `OPENAI_API_KEY`（或嵌入模型轮换），即使代码相同，向量路径结果也会漂移。
3. **捕获上限** —— 捕获的 `retrieved_slugs` 是去重集；它不保留内部排名元数据。两个工具可以返回相同的 slug 集但分数不同 —— Jaccard 会说 1.0，但按分数排序的下游消费者可能表现不同。

指标是**真实查询的回归警报**，而不是哈希检查。将它们与顶级回归的手动检查配对。

## 成本

快照中的每个 `query` 行都通过 OpenAI 嵌入查询字符串，以运行 `hybridSearch` 的向量部分。成本与正常的 `gbrain query` 调用相同 —— OpenAI 标价的 text-embedding-3-large，在单个重放行内分批处理。

如果您在本地迭代并且不想每次更改付费，请使用 `--limit 50` 限制重放的行数。最近的 50 行通常足以捕捉方向；在最终合并前运行时展开。

```bash
# 迭代模式 —— 最近的 50 个查询
gbrain eval replay --against baseline.ndjson --limit 50

# 合并前 —— 完整快照
gbrain eval replay --against baseline.ndjson --top-regressions 20
```

## CI 集成

```bash
gbrain eval replay --against baseline.ndjson --json > replay.json
jq -e '.summary.mean_jaccard >= 0.85' replay.json || exit 1
jq -e '.summary.top1_stability_rate >= 0.85' replay.json || exit 1
```

稳定的 JSON 形状（schema_version: 1）：

```json
{
  "schema_version": 1,
  "summary": {
    "rows_total": 247,
    "rows_replayed": 247,
    "rows_skipped": 0,
    "rows_errored": 0,
    "mean_jaccard": 0.927,
    "top1_stability_rate": 0.915,
    "mean_latency_delta_ms": 14,
    "rows_over_2x_latency": 0
  }
}
```

`--verbose` 添加 `results: [...]` 数组，每个重放行有一个条目（便于通过 jq 或笔记本进行更深入的分析）。

## 何时运行此命令

在合并任何接触以下内容的内容之前：

- `src/core/search/hybrid.ts`（RRF、融合、去重、两遍检索）
- `src/core/search/source-boost.ts` / `sql-ranking.ts`（每个源排名）
- `src/core/search/intent.ts`（自动详细分类）
- `src/core/search/expansion.ts`（Haiku 查询扩展）
- `src/core/search/dedup.ts`（跨页面结果折叠）
- `src/core/embedding.ts` 或任何嵌入模型交换
- `src/core/operations.ts` `query` 或 `search` 操作处理程序（捕获表面）
- `src/core/postgres-engine.ts` / `pglite-engine.ts` `searchKeyword` / `searchVector` SQL

跳过：仅架构迁移、文档更改、仅测试 PR、不触及检索的 CLI 人体工程学。

## 构建您自己的语料库

如果您还没有捕获的流量（新安装、合并前无法测试一周），您可以手动编写 NDJSON 文件：

```jsonl
{"schema_version":1,"id":1,"tool_name":"query","query":"who is alice","retrieved_slugs":["people/alice","people/alice-bio"],"expand_enabled":false,"detail":null,"latency_ms":0,"remote":false}
{"schema_version":1,"id":2,"tool_name":"search","query":"acme deal","retrieved_slugs":["deals/acme-seed","companies/acme"],"latency_ms":0,"remote":false}
```

然后运行 `gbrain eval replay --against handcrafted.ndjson` 确认权威 slug 返回。这是 BrainBench-Real 管道（针对实时捕获重放）和 BrainBench 固定 fixture 管道（使用兄弟 [gbrain-evals](https://github.com/garrytan/gbrain-evals) 语料库的 `gbrain eval --qrels`）之间的接缝。

## 关闭开关

两种禁用捕获的方法：

```bash
unset GBRAIN_CONTRIBUTOR_MODE             # 简单：只需取消设置环境变量
```

或者通过 `~/.gbrain/config.json` 强制关闭，无论环境变量如何：

```json
{"eval": {"capture": false}}
```

现有的 `eval_candidates` 行保留直到您执行 `gbrain eval prune --older-than 0d`（或直接删除表）。

## 失败模式

| 您看到什么 | 意味着什么 |
|---|---|
| `Mean Jaccard@k: 0.4`，顶级回归都在一个源目录中 | 该前缀的源提升或硬排除回归 |
| `Top-1 stability: 30%`，平均 Jaccard 仍然很高 | RRF 调优改变了排名顺序而没有改变集合 —— 重新调整 `rrfK` |
| `Mean latency Δ: +500ms`，Jaccard 很高 | 向量路径变慢；检查嵌入 API 或 HNSW 探针 |
| `rows_errored > 0` | 一个或多个查询抛出。检查人类输出中的前 3 个，或使用 `--json` 查看所有 `error_message` 字段 |
| 许多 `skipped: empty query` | 捕获在有人传递空 `query` 的行上运行 —— 检查为什么捕获了这些 |

## 公共基准：LongMemEval（v0.28.8）

`gbrain eval longmemeval` 直接针对 gbrain 的混合检索运行公共 [LongMemEval](https://huggingface.co/datasets/xiaowu0162/longmemeval) 基准。与 `eval replay` 不同的评估轴：公共数据集带有地面实况标签、端到端问答管道、每个问题的密封大脑。

```bash
# 下载数据集（在浏览器中访问 HF 页面；需要手动下载）。
# 将 longmemeval_oracle.json（或 _s.json）放在本地某处。

# 仅检索（无 LLM 答案生成，最快路径，无需 Anthropic 密钥）：
gbrain eval longmemeval ./longmemeval_oracle.json --limit 50 --retrieval-only \
  > /tmp/hypothesis.jsonl

# 完整管道（答案生成需要 Anthropic 密钥）：
gbrain eval longmemeval ./longmemeval_oracle.json --limit 50 \
  > /tmp/hypothesis.jsonl

# 使用 LongMemEval 发布的 evaluate_qa.py 评分（未捆绑 —— 需要
# OpenAI gpt-4o 根据他们的规范）：
python evaluate_qa.py /tmp/hypothesis.jsonl
```

### 架构（如果您要接触工具，请阅读此内容）

- 通过 `createBenchmarkBrain` + `withBenchmarkBrain` 每个基准运行一个内存中 PGLite。您的 `~/.gbrain` 永远不会被打开。
- 问题之间：`TRUNCATE` 在运行时枚举的 `pg_tables` 上，而不是硬编码列表 —— 架构迁移不会在问题之间静默泄漏数据。基础设施表（`sources`、`config`、`gbrain_cycle_locks`、`subagent_rate_leases`）在重置之间保留。
- 清理奇偶性：重用 `src/core/think/sanitize.ts` 中的 `INJECTION_PATTERNS`，因此添加新的注入模式会自动覆盖 takes 和基准测试。单一真相来源。
- 检索到的聊天内容包装在 `<chat_session id="..." date="...">` 框架中；答案生成系统提示声明内容不可信。与 `<take>` 框架相同的姿态。
- LLM 注入接缝：`runEvalLongMemEval(args, {client?: ThinkLLMClient})`。测试存根客户端，因此完整管道可以在没有任何 API 密钥的情况下密封运行。

### 标志

| 标志 | 默认值 | 用途 |
|---|---|---|
| `--limit N` | 运行全部 | 限制问题数量（快速迭代） |
| `--retrieval-only` | 关闭 | 发出检索到的块；无 LLM 答案生成 |
| `--keyword-only` | 关闭 | 禁用向量路径（调试检索问题） |
| `--expansion` | **关闭** | 多查询扩展。默认关闭以确保确定性（无每个查询的 Haiku 调用）。传入以选择加入。 |
| `--top-k K` | 10 | 检索深度 |
| `--model M` | resolved | 默认通过 `resolveModel()` 6 层链解析（`models.eval.longmemeval` 配置键） |
| `--output FILE` | stdout | 将假设 JSONL 写入文件而不是 stdout |

### 数字

p50 25.9ms / p99 30.3ms 在 Apple Silicon 上进行热重置+导入+搜索（根据 `test/eval-longmemeval.test.ts` 性能门）。每个问题的成本远低于 500ms 速度门。500 个问题 = ~13s 开销加上您的检索和 LLM 延迟。