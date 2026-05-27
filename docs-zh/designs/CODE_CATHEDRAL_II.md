# Code Cathedral II — v0.20.0 设计文档

**状态：** 已接受。CEO + 工程师 + 2 次 codex 审核通过（2026-04-24）。共吸收 16 个跨模型发现：7 个 codex pass 1（结构前提条件）+ 6 个 codex pass 2（吸收错误，包括 CHUNKER_VERSION silent-no-op gate 和入边失效）+ 3 个 eng-review 架构决策。DX 审查建议在 Layer 8（新 CLI 表面）之后、发布之前进行。
**取代：** Cathedral I（计划 v0.18.0–v0.19.0 代码索引，v0.19.0 发布）。
**模式：** SCOPE EXPANSION（用户明确："I want the best code search in the world"）。
**规模：** 14 个可二分的层，约 20–25 CC 小时，3–5 人周。一个 schema 迁移，带有分割边表（`code_edges_chunk` + `code_edges_symbol`）。通过 `CHUNKER_VERSION` bump（下次同步时自动）+ 显式 `gbrain reindex-code` 命令进行回填。

## 为什么是 v0.20.0

v0.19.0 发布了代码索引：tree-sitter chunker、29 种活动语言、符号列、正向文档↔实现链接、增量嵌入缓存、BrainBench 代码类别。四个 Cathedral-I 项目在发布期间被推迟：`query --lang` 过滤器、`sync --all` 成本预览、markdown fence 提取、反向扫描文档↔实现回填。

Cathedral II 是对这四个项目的承诺兑现版本，捆绑了使 gbrain 成为*最佳*代码搜索的飞跃：结构边（调用图 + 引用 + 导入 + 继承）、父作用域捕获、文档注释 FTS 绑定和两遍检索。不再是 grep 级别的代码检索。

## 10x 飞跃

今天：代理问"hybrid search 如何处理 N+1？" → 获取 `hybrid.ts` 的 3 个 prose chunks。

Cathedral II：相同查询返回锚点函数 + 其 3 个调用者 + 其 2 个被调用者 + 其 JSDoc + `/docs` 中引用它的指南 + 测试它的测试文件 + 父作用域链。一次遍历。代码感知大脑。

## 范围（5 层 + Layer 0 前提条件，14 个可二分的层提交）

### Tier 0 — 前提条件（codex 在语音之外提出）

**0a. 文件分类扩展。** `sync.ts:35` 当前仅将 9 种扩展名分类为代码（TS、JS、Python、Go、Rust、Ruby、Java、C、C++）。Cathedral II 的 B1 发布 165 个延迟加载语法，因此分类器需要接受 chunker 可以处理的任何扩展名。还重新排序 `detectCodeLanguage`，使 Magika（B2）作为无扩展名文件的回退运行，而不是在 null-return gate 之后。

**0b. Chunk-grain FTS。** 当前关键字搜索位于 `pages.search_vector`。在 chunk 级别添加文档注释或两遍锚定对页面粒度原语没有排名影响。Layer 0b 添加 `content_chunks.search_vector`，带有从限定符号名 + 文档注释（权重 A）和 chunk_text（权重 B）构建的触发器，并重写 `searchKeyword` 以直接对 chunks 进行排名。页面级 search_vector 保留用于标题密集型搜索。

两个 Layer 0 项目都是 10x 飞跃实际移动检索指标的前提条件。

### Tier A — 结构边（10x 飞跃）

**A1. 调用图 + 引用提取，带有限定符号标识。** `importCodeFile` 时的每语言 tree-sitter 查询捕获：

- `calls` — 函数调用站点
- `imports` — 模块依赖
- `extends` / `implements` — 类型层次结构
- `mixes_in` — Ruby `include`/`extend`/`prepend`
- `type_refs` — 参数 + 返回类型使用
- `declares` — chunk 拥有符号定义

**所有 8 种语言的限定符号标识。** `parent_symbol_path`（A3）是作用域的真相来源；边使用从它构建的限定名。示例：`Admin::UsersController#render`（Ruby 实例）、`Admin::UsersController.find_all`（Ruby 单例）、`admin.users_controller.UsersController.render`（Python）、`(*UsersController).Render`（Go）、`users::UsersController::render`（Rust）、`com.acme.admin.UsersController.render`（Java）。每语言分隔符 + 方法/类方法区分。Ruby 在排名器中完整发布（CLI + A2 两遍）—— 没有延迟。

**分割 schema（两个表，不是一个多态）：**
```sql
CREATE TABLE code_edges_chunk (
  from_chunk_id INTEGER NOT NULL REFERENCES content_chunks(id) ON DELETE CASCADE,
  to_chunk_id   INTEGER NOT NULL REFERENCES content_chunks(id) ON DELETE CASCADE,
  from_symbol_qualified TEXT NOT NULL,
  to_symbol_qualified   TEXT NOT NULL,
  edge_type     TEXT NOT NULL,
  source_id     TEXT REFERENCES sources(id) ON DELETE CASCADE,
  UNIQUE (from_chunk_id, to_chunk_id, edge_type)
);
CREATE TABLE code_edges_symbol (
  from_chunk_id INTEGER NOT NULL REFERENCES content_chunks(id) ON DELETE CASCADE,
  from_symbol_qualified TEXT NOT NULL,
  to_symbol_qualified   TEXT NOT NULL,
  edge_type     TEXT NOT NULL,
  source_id     TEXT REFERENCES sources(id) ON DELETE CASCADE,
  UNIQUE (from_chunk_id, to_symbol_qualified, edge_type)
);
```
`code_edges_chunk` = 已解析（两个端点已知）。`code_edges_symbol` = 未解析（目标符号按限定名存在，定义 chunk 尚未看到）。从 symbol→chunk 表的升级在后续导入时发生。`source_id` 是与实际 `sources.id` 类型匹配的 TEXT。

**已发布语言：** TypeScript、TSX、JavaScript、Ruby、Python、Go、Rust、Java（8 种语言，约占真实大脑代码的 85%）。其他语言正常分块（通过 B1 延迟加载）但在 v0.20.0 中不发出边 —— 扩展是每种语言一个查询文件 + 分隔符配置，可作为小的后续 PR 发布。

**A2. 两遍检索。** 当前：关键字 + 向量 → RRF → 去重。新：关键字 + 向量 → 锚点集 → 在 `code_edges_chunk` 上扩展 1–2 跳，带有结构距离衰减 → 混合到 RRF 中。

**在所有情况下默认关闭。** 仅通过 `--walk-depth N` 或 `--near-symbol <name>` 选择加入。精确符号匹配自动开启不安全（符号名跨文件冲突）。邻居上限每跳 50，深度上限 2。去重的每页上限（当前为 2）在遍历时提升到 `min(10, walkDepth × 5)`，因此来自一个文件的结构邻居不会被裁剪。距离衰减：扩展邻居 RRF 贡献上的 `1/(1 + hop)`。

**A3. 父作用域捕获 + 嵌套 chunk 发射。** 两部分：

*Part 1:* 嵌套符号在 `content_chunks` 上获得 `parent_symbol_path text[]`。嵌入到 chunk 头部：`[TypeScript] src/foo.ts:42-58 function formatResult (in BrainEngine.searchKeyword)`。作用域流入嵌入。双重用途：驱动 A1 的限定符号标识。

*Part 2:* 扩展 `splitLargeNode` 以将嵌套函数/方法/内部类作为自己的 chunks 发出。当前的 chunker 是顶级节点导向的 —— `class Foo { method1() {} method2() {} }` 发出一个 chunk。顶级节点上的 parent_symbol_path 为空（顶级之上没有父节点），因此没有子顶级 chunks，A3 没有贡献。Part 2 使作用域注释承载负载。

**A4. 文档注释 → 符号绑定。** 前导 AST 注释提取到 `doc_comment text`。以 FTS 权重 `'A'` 落在**chunk-grain** search_vector 上（Layer 0b 前提条件）。自然语言查询将文档字符串匹配排名高于正文且低于标题。根据 Postgres FTS 权重约定，`'A' > 'B' > 'C' > 'D'`。

### Tier B — 覆盖范围（诚实的 Chonkie 等价）

**B1.** 延迟加载 tree-sitter-language-pack（约 165 种语言）。用清单 + 每进程解析器缓存替换 36 个已提交的 WASM。Cathedral I 承诺了这一点但没有交付 — Cathedral II 做到了。

**B2.** 无扩展名文件的 Magika 自动检测（Dockerfile、Makefile、`.envrc`）。约 1MB 捆绑资产。如果分类器加载失败，则回退到 null → 递归 chunker。

### Tier C — 代理 CLI 表面

- `query --lang <lang>` — 按 `content_chunks.language` 过滤
- `query --symbol-kind function|class|method|type|interface|enum` — 按 `symbol_type` 过滤
- `query --near-symbol <name> --depth 1..2` — 在已知符号处锚定的两遍检索
- `code-callers <symbol>` — 使用 A1 `calls` 边，反向
- `code-callees <symbol>` — 使用 A1 `calls` 边，正向

非 TTY 上所有自动 JSON。失败时的 `StructuredAgentError` 信封。`code-signature` 推迟到 v0.20.1（需要每语言类型捕获）。

### Tier D — 桥梁项目（Cathedral I 承诺）

**D1.** `sync --all` 成本预览。`estimateTokens` 从 `chunkers/code.ts` 提取到新的 `tokens.ts` 模块。在每源循环之前：遍历同步差异集，求和 tokens，计算 $ 估算。TTY + !json + !yes → 交互式 `[y/N]`。非 TTY 或 `--json` 或管道 → 发出 `ConfirmationRequired` 信封，退出 2。`--yes` 跳过。`--dry-run` 预览 + 退出 0。仅在 `--all` 上预览，不在单源上预览（DX 审查痛点是首次大型同步惊喜账单）。

**D2.** `importFromContent` 中的 Markdown fence 提取。`parseMarkdown` 后，迭代标记的词法分析器 tokens 以获取 `{type:'code', lang, text}`。将 fence 标签映射到语言。通过 `chunkCodeText` 分块每个 fence。持久化为 `chunk_source='fenced_code'`。每页最多 100 个 fences（DOS 防御）。每个 fence try/catch — 一个坏 fence 不会破坏页面导入。

**D3.** `reconcile-links` 批量命令。遍历 markdown 页面，每页调用现有的 v0.19.0 `extractCodeRefs`，发出 `addLink(md, code, ..., 'documents')` + 反向。`ON CONFLICT DO NOTHING` 处理幂等性。语句超时通过 `sql.begin` + `SET LOCAL` 作用域。进度报告器 + 最终摘要（边添加/存在/目标缺失）。尊重 `auto_link` 配置。

### Tier E — 评估、回填、诚实

**E1.** BrainBench 代码子类别：`call_graph_recall`（X 的调用者 → 预期集）、`parent_scope_coverage`（嵌套符号查询返回正确作用域）、`doc_comment_matching`（NL 查询将文档注释排名高于 prose）。针对 A1/A3/A4 漂移的回归门。

**E2.** 回填：schema 自动迁移（零成本）。**`CHUNKER_VERSION` 从 3 提升到 4** — 该常量被折叠到每个代码页面的 `content_hash` 中，因此升级时每个代码页面的哈希都会更改。下次 `gbrain sync` 不会因"git HEAD 未更改"而短路；它会重新分块每个代码文件。新的 `gbrain reindex-code [--source <id>] [--dry-run] [--yes] [--force]` 提供显式完整回填，带有成本预览（重用 D1 基础设施），`--force` 完全绕过 content_hash 跳过。用户控制何时付费；静默 no-op 路径已关闭。

**E3.** 诚实的 CHANGELOG。取消"Chonkie superset"框架。在前后运行 BrainBench 获取真实数字：150+ 语言加载（B1 后）、NL→code 查询的 MRR、调用图精度 P@1、symbol_name 查询的 P@k、5K 文件仓库的同步成本预览。用可运行命令支持每个声明。

## 实现顺序（14 层，post-codex）

1. **0a** — 文件分类扩展（sync.ts:35）+ Magika 重新排序为回退
2. **0b** — Chunk-grain FTS（content_chunks.search_vector + 触发器 + searchKeyword 块级重写）
3. **Foundation** — schema 迁移（分割边表，content_chunks 上的限定名字段）+ 引擎方法存根 + 类型
4. **B1** — 延迟加载语法清单 + bun --compile 保护
5. **A1** — 边提取器 + 8 个每语言查询文件 + 限定符号标识 + 测试
6. **A3** — 父作用域列 + 文档注释列 + splitLargeNode 嵌套 chunk 发射
7. **A4** — chunk-grain search_vector 上的文档注释 FTS 权重 A
8. **A2** — 两遍检索，默认关闭，仅选择加入；遍历时去重上限提升
9. **D tier bundled** — 成本预览 + fence 提取 + reconcile-links
10. **B2** — Magika 自动检测
11. **C tier** — 5 个 CLI 表面
12. **E1** — BrainBench 子类别 + CHUNKER_VERSION 3→4 bump
13. **E2** — `reindex-code` 带 `--force` + 迁移编排器带回填提示阶段
14. **E3 + release** — 诚实的 CHANGELOG + 文档 + 迁移技能 + `/ship`

## 大小和成本

- Diff：约 5500–6500 行（约 2.5x v0.19.0 post-codex 扩展）
- 测试：约 2000 行（8 种语言 × 限定名 + 边提取 fixtures + Layer 0b FTS 迁移测试）
- 文件：约 36 个新文件，约 25 个修改文件
- CC 时间：约 20–25 小时专注（pre-codex 为 14–18；Layer 0a/0b + 8 种语言的限定标识 + 嵌套 chunk 发射 + CHUNKER_VERSION bump 层 +6h）
- 人力等效：3–5 周
- 升级 v0.19.0 用户的首次同步成本增加：升级后的首次同步时每个代码页面重新分块（CHUNKER_VERSION bump 强制失效）。用户运行 `gbrain reindex-code --dry-run` 获取成本预览，然后 `--yes` 或接受随时间逐步回填，因为文件会变化。
- 回填后的每日 autopilot 成本：不变（边在 chunk 时提取，无每查询 LLM）

## 风险和缓解措施

1. **实时 Postgres 上的 Schema 迁移。** 发布前针对生产形状数据库测试。v0.12.0 JSONB 事件是警示。
2. **每语言 tree-sitter 查询很繁琐。** 每种语言手动验证边集 fixtures。Ruby 获得额外覆盖以处理动态调度假阴性。
3. **两遍检索回归。** 散文默认关闭。BrainBench Cat 1 必须在发布前显示无回归。
4. **回填形状（G1 已解决）。** 三层可组合：schema-auto 迁移空列（零成本）。延迟触达捕获 80% 随时间推移（零成本）。显式 `reindex-code` 带成本预览，供希望立即获得全部好处的用户使用。无惊喜账单。
5. **Magika bundle（G2 已解决）。** +1MB 资产，`bun --compile` 保护扩展。如果捆绑在实现后期出现 bug，B2 是唯一可以回退到 v0.20.1 而不阻塞大教堂的层 — 它在 Layer 8 是自包含的。
6. **高扇出符号。** `console.log` 风格的符号有 100K 调用者。邻居上限 50，深度上限 2。需要混沌测试 fixture。

## 审查门

- CEO 审查（Cathedral II）—— CLEARED 2026-04-24
- Outside voice（codex）—— 在 Cathedral II CEO 审查期间运行
- `/plan-devex-review` —— 下一个（根据用户请求，5 个新 CLI 表面 + reindex-code 需要 DX 抛光审查在 eng 之前）
- `/plan-eng-review` —— 实施开始前必需
- `/review` + `/codex review` —— `/ship` 之前必需

## 推迟到后续大教堂的内容

- **C6** `code-signature "(A, B) => C"` — 每语言类型捕获。v0.20.1。
- **8 种已发布语言之外的调用图语言** — PHP、Swift、Kotlin、Scala、C#、C++、Elixir 等。每种语言一个小 PR。
- **LSP 集成** 用于实时精度。v0.22+ 大教堂。
- **Code-tour generator**（Cathedral I T1）。
- **嵌入前的私有代码编辑**（Cathedral I T3）。
- **`gbrain doctor --chunker-debug`** AST dump。