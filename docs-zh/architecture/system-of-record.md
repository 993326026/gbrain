# System of record

**GitHub repo（markdown + frontmatter）是记录系统。
Postgres/PGLite 数据库是派生缓存。我们不备份数据库 —— 我们从 repo 重建它。**

本文档是该契约的规范参考。每个写入用户知识状态的代码路径都应符合此处描述的模式。`scripts/check-system-of-record.sh` 处的 CI 门以编程方式强制执行。

## 为什么这很重要

数据库是 markdown 内容上的派生索引。它的存在是为了使搜索更快，对嵌入相似的声明进行去重，实现跨页面图。这些数据都不是不可替代的 —— 只要 markdown 完好无损，`gbrain sync && gbrain extract all` 就能从头重建整个数据库。

这意味着：

- **灾难恢复只需一个命令。** 如果您的数据库卷损坏，如果 Postgres 自我损坏，如果 PGLite 的 WASM 锁卡住 —— 您不需要备份。您擦除数据库，从您的 brain repo 重新导入，派生状态会重新生成。v0.32.3 发布 `gbrain rebuild --confirm-destructive` 作为文档化的单行命令。
- **多机器同步就是 git。** 您的 brain 是一个 repo。从一台机器推送，从另一台机器拉取，第二台机器的数据库会在下次同步时重建。无需"备份数据库"步骤。
- **隐私在您手中。** 敏感实体页面可以被 gitignore（通过 `gbrain.yml` `db_only` 路径或每页），它们保留在磁盘上但不在 git 中。隔离墙尊重您在页面级别做出的任何 git 跟踪选择。
- **跨代理协作成为可能。** 多个代理可以写入同一个 brain，因为隔离墙是合并点，而不是数据库。Git 以 git 处理并发编辑的方式处理并发编辑。

## 三个类别

gbrain schema 中的每个表恰好属于三个类别之一。类别决定了它在灾难恢复期间如何重建。

### FS-canonical（markdown 是真相的来源）

这些是用户创作的知识。数据库行是 markdown 上的派生索引 —— 擦除表，`gbrain extract` 会以相同方式重建它。CI 门防止直接数据库写入偏离 markdown 契约。

| 类别 | 如何存储在 markdown 中 | 派生数据库表 | Reconciler |
|---|---|---|---|
| **Takes**（包括 hunches, bets） | `## Takes` 围栏表，位于 `<!--- gbrain:takes:begin -->` / `:end -->` 标记之间 | `takes` | `extract takes` |
| **Facts** | `## Facts` 围栏表，位于 `<!--- gbrain:facts:begin -->` / `:end -->` 标记之间 | `facts` | `extract_facts` 周期阶段 |
| **Links** | markdown 正文中的内联 `[text](slug)` / `[[slug]]` + frontmatter `direction: incoming` | `links` | `extract links` |
| **Timeline** | `<!-- timeline -->` 标记后的 `## Timeline` 部分 | `timeline_entries` | `extract timeline` |
| **Tags** | Frontmatter `tags:` YAML 数组 | `tags` | `importFromFile`（导入时每页协调） |
| **emotional_weight** | 从 takes + tags 重新计算 | `pages.emotional_weight`（信号列） | `recompute_emotional_weight` 周期阶段 |
| **synthesis_evidence** | synthesis 页面内的 `takes` 行 FK（`slug#N`） | `synthesis_evidence` | `extract takes`（传递性） |

### 从 FS 派生但非用户创作

这些包含从 markdown 自动重建但不是用户直接作为 markdown 创作的派生状态。分块器 + 嵌入器在导入时重建这些。

| 表 | 来源 | 说明 |
|---|---|---|
| `pages` | 整个 markdown 文件 | 每个文件一行；`compiled_truth` + `frontmatter` 来自解析 |
| `content_chunks` | `pages.compiled_truth` 经过分块器剥离后 | 内容哈希更改时重新分块；通过配置的模型嵌入 |
| `page_versions` | 每次 `pages` UPDATE | 审计历史；原则上可重建但实际上不可 |

### DB-only by design（命名异常）

这些包含故意不在 repo 中的运行时/基础设施状态。架构规则仍然适用 —— 这些不是"用户知识" —— 但它们在设计上是 DB-only 的。

| 类别 | 为什么可以是 DB-only |
|---|---|
| `raw_data` | Webhook/转录 sidecar；不是用户创作的知识。 |
| `subagent_messages` / `subagent_tool_executions` / `subagent_rate_leases` | 运行时作业状态。仅重放，不是持久知识。 |
| `oauth_clients` / `oauth_tokens` / `access_tokens` | 凭据。根据定义不在源代码控制中。 |
| `mcp_request_log` | 审计跟踪。设计上是易变的。 |
| `minion_jobs` / `minion_inbox` / `minion_attachments` | 作业队列。重启时重新入队或丢弃。 |
| `eval_candidates` / `eval_capture_failures` | 贡献者模式开发循环；选择加入捕获。 |
| `dream_verdicts` | 廉价的裁决缓存。通过重新运行 Haiku 可重建。 |
| `gbrain_cycle_locks` / migration ledger | 基础设施。 |
| `config`（某些键） | 站点本地路由配置（例如 `sync.repo_path`）。 |

包含用户知识的新派生表**必须**首先在 FS 中落地。
如果您想添加一个"现在是 DB-only"的表，结构性问题是：它是否属于这个 DB-only-by-design 列表？如果不是，它是 FS-canonical 的，需要一个围栏（或 frontmatter 字段）加上一个 reconciler。

## 隐私边界

围栏中的私有知识仍然存在于 markdown 文件中。如果用户将页面提交到 git，私有数据也会进入 git。这是现有的操作模型 —— 我们不推断 git 策略。

对于不受信任的读取器（远程 MCP、子代理），v0.32.2 版本提供了三层剥离：

1. **Layer A（分块器）：** `src/core/chunkers/recursive.ts` 在分块前调用 `stripFactsFence({keepVisibility: ['world']})` + `stripTakesFence`。私有事实文本永远不会到达 `content_chunks.chunk_text`、嵌入或搜索结果。
2. **Layer B（get_page）：** 当 `ctx.remote === true` 时，响应正文会剥离两个围栏（来自 facts 的私有行；整个 takes 围栏）。本地 CLI（`ctx.remote === false`）看到完整围栏。
3. **Layer C（git 跟踪）：** 用户决定是否提交实体页面。`gbrain.yml` `db_only` 路径自动被 gitignored；通过用户的正常 git 工作流进行每页选择。

对于普遍私有的实体（朋友的名字、投资者的内部笔记），在 `gbrain.yml` 中将实体页面的目录标记为 `db_only`。文件保留在磁盘上但永远不会进入 git。

## 遗忘契约

`gbrain forget <id>` 和 MCP `forget_fact` 操作使用删除线 + `valid_until = today` + `context: "forgotten: <reason>"` 重写围栏行。数据库的 `expired_at = valid_until + now()` 派生在每次重建时重建遗忘状态，因为围栏是规范的。

删除线有两种语义，通过上下文区分：

- `~~claim~~` + `context: "superseded by #N"` → 行被同一围栏中的较新行替换
- `~~claim~~` + `context: "forgotten: <reason>"` → 行通过 forget 操作撤回

两种编码都将行保留在 markdown 中用于审计历史。要永久删除事实，请直接在 markdown 中编辑围栏并删除该行。下一个 `extract_facts` 周期会擦除数据库行。

## 灾难恢复

规则做出的承诺：

```bash
# 快照现有内容
gbrain stats > /tmp/before.txt

# 擦除并重建
gbrain rebuild --confirm-destructive   # v0.32.3 — 删除派生表
                                       # (pages + content_chunks 在 CASCADE-safe 设计中幸存)
                                       # 或 v0.32.2 手动方式：
psql -c 'DELETE FROM facts; DELETE FROM takes; DELETE FROM links; DELETE FROM timeline_entries;'
gbrain sync
gbrain extract all

# 计数匹配
gbrain stats > /tmp/after.txt
diff /tmp/before.txt /tmp/after.txt
```

`test/e2e/system-of-record-invariant.test.ts` 处的不变量 E2E 测试在每次 CI 运行时执行此精确流程。

## 新代码规则

添加新的用户知识类别时：

1. **定义 markdown 形状。** 围栏（`<!--- gbrain:NAME:begin --> ... :end -->` 表）或 frontmatter 字段。
2. **构建解析器**，从 markdown 生成结构化数据。请参阅 `src/core/fence-shared.ts` 获取共享原语。
3. **构建写入器**，支持往返：解析 + 编辑 + 渲染产生相同输入的字节相同的 markdown。
4. **添加引擎方法**，接受解析数据并标记派生表。该方法在 CI 门的禁止直接调用列表中获得条目。
5. **添加 reconciler：** 一个周期阶段，遍历页面，解析围栏，并从头重建派生表。Reconciler 是引擎方法的唯一合法调用站点；`// gbrain-allow-direct-insert: <reason>` 显式注释它。
6. **在 `test/e2e/system-of-record-invariant.test.ts` 中添加往返测试**，证明 DELETE + reconcile 字节相同地重建表。

`scripts/check-system-of-record.sh` 处的 CI 门会失败任何在 reconciler / migration 层之外添加新的派生表写入器直接调用而没有显式允许列表注释的 PR。

## 相关

- `~/.claude/plans/system-instruction-you-are-working-expressive-pony.md`
  — v0.32.2 设计计划（决策 D1-D22 + Q1-Q8，Codex round 1 和 round 2 发现）
- `skills/migrations/v0.32.2.md` — 面向代理的迁移指南
- `CHANGELOG.md` v0.32.2 条目 — 发布宣言
- `scripts/check-system-of-record.sh` — 强制执行规则的 CI 门