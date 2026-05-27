# 可插拔引擎架构

## 设计理念

所有 GBrain 操作都通过 `BrainEngine` 执行。引擎是"大脑能做什么"和"如何存储"之间的契约。更换引擎，保持其他一切不变。

v0 版本发布了由 Supabase 支持的 `PostgresEngine`。v0.7 添加了 `PGLiteEngine` —— 通过 WASM 嵌入的 PostgreSQL 17.5（@electric-sql/pglite），零配置默认选项。该接口设计允许 `DuckDBEngine`、`TursoEngine` 或任何自定义后端无需修改 CLI、MCP 服务器、技能或任何消费代码即可插入。

## 为什么这很重要

不同用户有不同的限制：

| 用户类型 | 需求 | 最佳引擎 |
|----------|------|----------|
| 入门用户 | 零配置、无需账户、无需服务器 | PGLiteEngine（v0.7 起默认） |
| 高级用户 | 世界级搜索、7000+ 页面、零运维 | PostgresEngine + Supabase |
| 开源爱好者 | 单文件、无服务器、Git 友好 | PGLiteEngine |
| 团队/企业 | 多用户、RLS、审计追踪 | PostgresEngine + 自托管 |
| 研究者 | 分析、批量导出、嵌入分析 | DuckDBEngine（未来） |
| 边缘/移动 | 离线优先、稍后同步 | PGLiteEngine + 同步（未来） |

引擎接口意味着我们不必做出选择。PGLite 是零摩擦默认选项。Supabase 是生产级扩展路径。`gbrain migrate --to supabase/pglite` 在它们之间迁移。

## 接口定义

```typescript
// src/core/engine.ts

export interface BrainEngine {
  // 生命周期
  connect(config: EngineConfig): Promise<void>;
  disconnect(): Promise<void>;
  initSchema(): Promise<void>;
  transaction<T>(fn: (engine: BrainEngine) => Promise<T>): Promise<T>;

  // 页面 CRUD
  getPage(slug: string): Promise<Page | null>;
  putPage(slug: string, page: PageInput): Promise<Page>;
  deletePage(slug: string): Promise<void>;
  listPages(filters: PageFilters): Promise<Page[]>;

  // 搜索
  searchKeyword(query: string, opts?: SearchOpts): Promise<SearchResult[]>;
  searchVector(embedding: Float32Array, opts?: SearchOpts): Promise<SearchResult[]>;

  // 块
  upsertChunks(slug: string, chunks: ChunkInput[]): Promise<void>;
  getChunks(slug: string): Promise<Chunk[]>;

  // 链接
  addLink(from: string, to: string, context?: string, linkType?: string): Promise<void>;
  removeLink(from: string, to: string): Promise<void>;
  getLinks(slug: string): Promise<Link[]>;
  getBacklinks(slug: string): Promise<Link[]>;
  traverseGraph(slug: string, depth?: number): Promise<GraphNode[]>;

  // 标签
  addTag(slug: string, tag: string): Promise<void>;
  removeTag(slug: string, tag: string): Promise<void>;
  getTags(slug: string): Promise<string[]>;

  // 时间线
  addTimelineEntry(slug: string, entry: TimelineInput): Promise<void>;
  getTimeline(slug: string, opts?: TimelineOpts): Promise<TimelineEntry[]>;

  // 原始数据
  putRawData(slug: string, source: string, data: object): Promise<void>;
  getRawData(slug: string, source?: string): Promise<RawData[]>;

  // 版本
  createVersion(slug: string): Promise<PageVersion>;
  getVersions(slug: string): Promise<PageVersion[]>;
  revertToVersion(slug: string, versionId: number): Promise<void>;

  // 统计 + 健康检查
  getStats(): Promise<BrainStats>;
  getHealth(): Promise<BrainHealth>;

  // 摄入日志
  logIngest(entry: IngestLogInput): Promise<void>;
  getIngestLog(opts?: IngestLogOpts): Promise<IngestLogEntry[]>;

  // 配置
  getConfig(key: string): Promise<string | null>;
  setConfig(key: string, value: string): Promise<void>;

  // 迁移 + 高级功能（v0.7 添加）
  runMigration(sql: string): Promise<void>;
  getChunksWithEmbeddings(slug: string): Promise<ChunkWithEmbedding[]>;
}
```

### 关键设计选择

**基于 Slug 的 API，而非基于 ID**。每个方法都接受 slug，而不是数字 ID。引擎内部将 slug 解析为 ID。这保持接口的可移植性...slug 是字符串，ID 是数据库特定的。

**嵌入不在引擎中**。引擎存储嵌入并按向量搜索，但不生成嵌入。`src/core/embedding.ts` 处理嵌入。这是有意设计的：嵌入是外部 API 调用（OpenAI），不是存储问题。所有引擎共享相同的嵌入服务。

**分块不在引擎中**。同样的逻辑。`src/core/chunkers/` 处理分块。引擎存储和检索块。所有引擎共享相同的分块器。

**搜索返回 `SearchResult[]`，而非原始行**。引擎负责自己的搜索实现（tsvector vs FTS5，pgvector vs sqlite-vss），但必须返回统一的结果类型。RRF 融合和去重在引擎之上的 `src/core/search/hybrid.ts` 中进行。

**`traverseGraph` 存在但引擎特定**。Postgres 使用递归 CTE。SQLite 将使用带深度跟踪的循环。接口相同：给我一个 slug 和最大深度，返回图。

## 跨引擎搜索工作原理

```
                        +-------------------+
                        |  hybrid.ts        |
                        |  (RRF 融合 +     |
                        |   去重，共享)    |
                        +--------+----------+
                                 |
                    +------------+------------+
                    |                         |
           +--------v--------+       +--------v--------+
           | engine.search   |       | engine.search   |
           |   Keyword()     |       |   Vector()      |
           +-----------------+       +-----------------+
                    |                         |
        +-----------+-----------+   +---------+---------+
        |                       |   |                   |
+-------v-------+  +-------v---+   +-------v---+  +----v--------+
| Postgres:     |  | PGLite:   |   | Postgres: |  | PGLite:     |
| tsvector +    |  | tsvector +|   | pgvector  |  | pgvector    |
| ts_rank +     |  | ts_rank   |   | HNSW      |  | HNSW        |
| websearch_to_ |  | (相同 SQL)|   | cosine    |  | cosine      |
| tsquery       |  |           |   |           |  | (相同 SQL)  |
+---------------+  +-----------+   +-----------+  +-------------+
```

RRF 融合、多查询扩展和四层去重是引擎无关的。它们在 `SearchResult[]` 数组上操作。只有原始关键词和向量搜索是引擎特定的。

## PostgresEngine（v0，已发布）

**依赖**: `postgres` (porsager/postgres), `pgvector`

**使用的 Postgres 特定功能**:
- `tsvector` + `GIN` 索引用于全文搜索，带 `ts_rank` 加权
- `pgvector` HNSW 索引用于余弦相似度向量搜索
- `pg_trgm` + `GIN` 用于模糊 slug 解析
- 递归 CTE 用于图遍历
- 基于触发器的 search_vector（跨 pages + timeline_entries）
- JSONB 用于 frontmatter，带 GIN 索引
- 通过 Supabase Supavisor 进行连接池（端口 6543）

**托管**: Supabase Pro（$25/月）。零运维。内置 pgvector 的托管 Postgres。

**为什么 v0 不自托管**: 大脑应该是 Agent 使用的基础设施，而不是你需要维护的东西。Docker 自托管 Postgres 是欢迎的社区 PR，但 v0 优化零运维。

## PGLiteEngine（v0.7，已发布）

**依赖**: `@electric-sql/pglite`（v0.4.4+）

**是什么**: 通过 ElectricSQL 的 PGLite 编译为 WASM 的嵌入式 Postgres 17.5。进程内运行，无需服务器，无需 Docker，无需账户。与 PostgresEngine 使用相同的 SQL —— 不是单独的方言。实现了全部 37 个 BrainEngine 方法。

**PGLite 特定细节**:
- 使用 `pglite-schema.ts` 进行 DDL（pgvector 扩展、pg_trgm、触发器、索引）
- 全程使用参数化查询（共享工具在 `src/core/utils.ts`）
- 未设置 `OPENAI_API_KEY` 时的混合搜索关键词回退
- 数据存储在 `~/.gbrain/brain.db`（可配置）
- pgvector HNSW 索引用于余弦相似度向量搜索（与 Postgres 相同）
- tsvector + ts_rank 用于全文搜索（与 Postgres 相同）
- pg_trgm 用于模糊 slug 解析（与 Postgres 相同）

**何时使用 PGLite vs Postgres**:

| 因素 | PGLite | PostgresEngine + Supabase |
|------|--------|--------------------------|
| 设置 | `gbrain init`（零配置） | 账户 + 连接字符串 |
| 规模 | 适合 < 1,000 文件 | 生产验证 10K+ |
| 多设备 | 仅单机 | 通过远程 MCP 任意设备 |
| 成本 | 免费 | Supabase Pro（$25/月） |
| 并发 | 单进程 | 连接池 |
| 备份 | 手动（文件复制） | Supabase 托管 |

**迁移**: `gbrain migrate --to supabase` 导出所有内容（页面、块、嵌入、链接、标签、时间线）并导入到 Supabase。`gbrain migrate --to pglite` 反向迁移。双向，无损。

## 添加新引擎

1. 创建 `src/core/<name>-engine.ts` 实现 `BrainEngine`
2. 添加到 `src/core/engine-factory.ts` 中的引擎工厂：
   ```typescript
   export function createEngine(type: string): BrainEngine {
     switch (type) {
       case 'pglite': return new PGLiteEngine();
       case 'postgres': return new PostgresEngine();
       case 'myengine': return new MyEngine();
       default: throw new Error(`Unknown engine: ${type}`);
     }
   }
   ```
   工厂使用动态导入，因此引擎仅在选择时加载。
3. 在 `~/.gbrain/config.json` 中存储引擎类型：`{ "engine": "myengine", ... }`
4. 添加测试。测试套件应尽可能引擎无关...相同的测试用例，不同的引擎构造函数。
5. 在此文件中记录 + 在 `docs/` 中添加设计文档

### 不需要修改的内容

- `src/cli.ts`（调度到引擎，不知道具体是哪个）
- `src/mcp/server.ts`（相同）
- `src/core/chunkers/*`（跨引擎共享）
- `src/core/embedding.ts`（跨引擎共享）
- `src/core/search/hybrid.ts`、`expansion.ts`、`dedup.ts`（共享，在 SearchResult[] 上操作）
- `skills/*`（胖 markdown，引擎无关）

### 需要实现的内容

`BrainEngine` 中的每个方法。完整接口。无可选方法，无功能标志。如果你的引擎不能进行向量搜索（例如纯文本引擎），实现 `searchVector` 返回 `[]` 并记录限制。

## 能力矩阵

| 能力 | PostgresEngine | PGLiteEngine | 说明 |
|------|---------------|--------------|------|
| CRUD | 完整 | 完整 | 相同 SQL |
| 关键词搜索 | tsvector + ts_rank | tsvector + ts_rank | 相同（真正的 Postgres） |
| 向量搜索 | pgvector HNSW | pgvector HNSW | 相同（真正的 Postgres） |
| 模糊 slug | pg_trgm | pg_trgm | 相同（真正的 Postgres） |
| 图遍历 | 递归 CTE | 递归 CTE | 相同 SQL |
| 事务 | 完整 ACID | 完整 ACID | 两者都支持 |
| JSONB 查询 | GIN 索引 | GIN 索引 | 相同 |
| 并发访问 | 连接池 | 单进程 | PGLite 限制 |
| 托管 | Supabase、自托管、Docker | 本地文件 | |
| 迁移方法 | runMigration, getChunksWithEmbeddings | 相同 | v0.7 添加 |

## 未来引擎想法

**TursoEngine**。libSQL（SQLite 分支），带嵌入式副本和 HTTP 边缘访问。将提供 SQLite 的简单性和云同步。对移动/边缘用例很有趣。

**DuckDBEngine**。分析工作负载。批量导出、嵌入分析、全脑统计。不适合 OLTP。可以作为 Postgres 的辅助引擎用于分析。

**自定义/远程**。接口足够清晰，有人可以构建由任何存储支持的引擎：Firestore、DynamoDB、REST API，甚至平面文件系统。接口不假设 SQL。

注意：原始 SQLite 引擎计划（`docs/SQLITE_ENGINE.md`）已被 PGLite 取代。PGLite 使用与 Postgres 相同的 SQL，消除了对使用 FTS5/sqlite-vss 转换的单独 SQLite 方言的需求。