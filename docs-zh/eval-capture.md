# Eval capture — NDJSON schema reference

**状态：** 从 v0.21.0 开始稳定。通过每行的 `schema_version` 进行版本控制；增量更改增加次要版本；删除是破坏性的 schema-v2。

**受众：** 下游消费者（主要是兄弟 [gbrain-evals](https://github.com/garrytan/gbrain-evals) 仓库），将捕获的真实查询作为 BrainBench-Real fixture 重放。

## 管道

```
MCP / CLI / subagent tool-bridge caller
     │
     ▼
src/core/operations.ts — query + search op handlers
     │
     │ (hybridSearch or searchKeyword)
     │
     ▼
{results, meta: HybridSearchMeta}                 ┌── captureEvalCandidate
     │                                             │    (fire-and-forget)
     ▼                                             │
return to caller                                   ▼
                                            scrubPii(query) ←── src/core/eval-capture-scrub.ts
                                                   │
                                                   ▼
                                           buildEvalCandidateInput
                                                   │
                                                   ▼
                                           engine.logEvalCandidate
                                                   │
                                    ┌──────────────┴──────────────┐
                                    │ success                     │ fail
                                    ▼                             ▼
                                INSERT into eval_candidates    engine.logEvalCaptureFailure
                                                                 (reason: db_down | rls_reject |
                                                                  check_violation |
                                                                  scrubber_exception | other)
```

## `gbrain eval export` — 消费者契约

```sh
gbrain eval export [--since DUR] [--limit N] [--tool query|search]
```

将 NDJSON 输出到 **stdout**。每个 `\n` 终止行一个 JSON 对象。stderr 接收进度心跳。每行以 `"schema_version": 1` 开头，因此向前兼容的解析器可以在 schema v2 上大声失败，而不是静默解析错误。

gbrain-evals 的典型用法：

```sh
# 快照最近一周的真实流量用于重放
gbrain eval export --since 7d > brainbench-real.ndjson
```

```sh
# 通过 jq 流式传输进行临时分析
gbrain eval export --tool query | jq -c 'select(.latency_ms > 500)'
```

## 行 schema (v1)

每个导出的行都有这个形状。JSON 输出中的字段顺序不保证；消费者**必须**按名称键控，而不是位置。

| 字段 | 类型 | 说明 |
|---|---|---|
| `schema_version` | number | v1 行上始终为 `1`。向前兼容门。 |
| `id` | number | 自增主键。跨导出稳定。 |
| `tool_name` | `"query"` \| `"search"` | 哪个 MCP 操作捕获了此行。 |
| `query` | string | **已由 `scrubPii` 进行 PII 清理**，除非 `eval.scrub_pii: false`。电子邮件/电话/SSN/Luhn 验证的信用卡/JWT/bearer tokens 替换为 `[REDACTED]`。最大长度 50KB（CHECK 强制）。 |
| `retrieved_slugs` | string[] | `SearchResult[]` 中返回的去重 slugs。 |
| `retrieved_chunk_ids` | number[] | 结果顺序中的每个块 ID（保留重复项 — 每个命中一个）。 |
| `source_ids` | string[] | 结果集中不同的 `sources.id` 值（v0.18 多源）。对于缺少该列的 v0.18 前行，为空。 |
| `expand_enabled` | boolean \| null | 调用者是否**请求**了 Haiku 扩展。`search` 为 `null`（无扩展概念）。 |
| `detail` | `"low"` \| `"medium"` \| `"high"` \| null | 调用者**请求**的详细级别。省略时为 `null`。 |
| `detail_resolved` | `"low"` \| `"medium"` \| `"high"` \| null | `hybridSearch` 在自动检测后**实际使用**的内容。当调用者和启发式都未分类时为 `null`。 |
| `vector_enabled` | boolean | 向量搜索是否实际运行。当 `OPENAI_API_KEY` 缺失或嵌入调用失败时为 `false`。**重放必须尊重这一点** — `false` 的行只执行了关键词路径。 |
| `expansion_applied` | boolean | Haiku 扩展是否实际产生了变体（不仅仅是"被请求"）。 |
| `latency_ms` | number | 操作处理程序的挂钟持续时间（包括捕获本身 — 可忽略，因为它是 fire-and-forget）。 |
| `remote` | boolean | MCP 调用者为 `true`（不受信任），本地 CLI 为 `false`。将"真实代理流量"与"操作员探测"分区。 |
| `job_id` | number \| null | 当调用者是子代理工具桥时为 `OperationContext.jobId`。MCP + CLI 为 null。 |
| `subagent_id` | number \| null | 子代理拥有的运行的 `OperationContext.subagentId`。 |
| `created_at` | string (ISO 8601) | 插入的 UTC 时间戳。 |

## 排序 + 确定性

`listEvalCandidates` 按 `created_at DESC, id DESC` 排序。同一毫秒插入在 `created_at` 上并列；`id DESC` 是稳定的平局决胜者。重放工具可以按顺序消费行并假设：
- 非重叠 `--since` 窗口的调用之间没有重复行
- 链接 `--since` 窗口的调用之间没有遗漏行（运行 1 的窗口结束是严格的上限，不是软游标）

## Schema 版本化承诺

- **v1（v0.21.0 发布）** — 本文档。上面列出的所有字段。
- **增量更改** 增加 gbrain 次要版本（v0.25.0、v0.23.0 …）并附带新的可选字段。基于已知字段的消费者忽略未知键并继续工作。
- **破坏性更改**（重命名、类型更改、删除）增加 `schema_version` 到 2。消费者**必须**在 `schema_version` 上分支以保持兼容。

## `eval_capture_failures` — 伴随审计表

不被 `gbrain eval export` 导出。通过 `gbrain doctor` 显示：

```sh
gbrain doctor   # 当过去 24h 失败 > 0 时警告
```

原因枚举（稳定）：`db_down` | `rls_reject` | `check_violation` | `scrubber_exception` | `other`。跨进程可见性是关键点 — `gbrain doctor` 在自己的进程中运行并直接读取表，因此进程内计数器不起作用。

## Config + CONTRIBUTOR_MODE

从 v0.25.0 开始，捕获**默认关闭**（早期草稿中对所有人开启）。开启的两种方式：

**路径 A — 环境变量（贡献者选择加入，常见情况）：**

```bash
export GBRAIN_CONTRIBUTOR_MODE=1     # 在 ~/.zshrc 或 ~/.bashrc 中
```

**路径 B — 显式配置（`~/.gbrain/config.json`，仅文件平面）：**

```json
{
  "engine": "postgres",
  "database_url": "...",
  "eval": {
    "capture": true,
    "scrub_pii": true
  }
}
```

解析顺序（最明确的获胜）：

1. 配置中的 `eval.capture: true` → 开启
2. 配置中的 `eval.capture: false` → 关闭（覆盖 CONTRIBUTOR_MODE=1）
3. `GBRAIN_CONTRIBUTOR_MODE === '1'` → 开启
4. 否则 → 关闭

`scrub_pii` 独立于捕获默认为 `true`。设置 `eval.scrub_pii: false` 以保留原始查询文本（仅当您控制大脑的分发时）。

`gbrain config set eval.capture false` **不工作** — 该命令写入数据库平面配置，而 MCP 服务器读取文件平面。直接编辑 JSON 或使用环境变量。