# GBrain v0: Postgres-Native 个人知识大脑

## 这是什么

GBrain 是一个编译智能系统。不是笔记应用。不是"与你的笔记聊天"。

每个页面都是一份智能评估。线上方：编译真相（你当前最好的理解，有新证据时重写）。线下方：时间线（仅追加证据轨迹）。AI 代理维护大脑。MCP 客户端查询它。智能存在于胖 markdown 技能中，而不是应用代码中。

核心洞察：大规模个人知识是智能问题，而不是存储问题。

## 为什么存在

一个 7,471 文件 / 2.3GB 的 markdown wiki 正在让 git 窒息。Git 在 wiki 风格使用中无法扩展到 ~5K 文件以上。编译真相 + 时间线模型（Karpathy 风格的知识页面）是正确的，但它下面需要一个真正的数据库。

已经有一个生产级 RAG 系统（Ruby on Rails、Postgres + pgvector），具有 3 层分块、带 RRF 的混合搜索、多查询扩展和 4 层去重。GBrain 将这些经过验证的模式移植到独立的 Bun + TypeScript 工具中。

## 知识模型

```
+--------------------------------------------------+
|  Page: concepts/do-things-that-dont-scale         |
|                                                   |
|  --- frontmatter (YAML) ---                       |
|  type: concept                                    |
|  tags: [startups, growth, pg-essay]               |
|                                                   |
|  === COMPILED TRUTH ===                           |
|  Current best understanding.                      |
|  Rewritten on new evidence.                       |
|  This is the "what we know now" section.          |
|                                                   |
|  ---                                              |
|                                                   |
|  === TIMELINE ===                                 |
|  Append-only evidence trail.                      |
|  - 2013-07-01: Published on paulgraham.com        |
|  - 2024-11-15: Referenced in batch kickoff talk   |
|  Never edited, only appended.                     |
+--------------------------------------------------+
          |                    |
          v                    v
  [Semantic chunks]     [Recursive chunks]
  (best quality for     (predictable format
   compiled truth)       for timeline)
          |                    |
          v                    v
     [Embeddings: text-embedding-3-large, 1536 dims]
          |
          v
  [HNSW index + tsvector + pg_trgm]
          |
          v
  [Hybrid search: vector + keyword + RRF fusion]
```

## 架构决策

### v0 技术栈

| 层 | 选择 | 原因 |
|-------|--------|-----|
| 数据库 | Postgres + pgvector | 经过验证的 RAG 模式，生产测试。世界级混合搜索。 |
| 托管 | Supabase Pro（$25/月） | 零运维。托管 Postgres、pgvector、连接池。8GB 存储。 |
| 运行时 | Bun + TypeScript | 与 GStack 生态系统一致。快速。编译为单个二进制文件。 |
| 嵌入 | OpenAI text-embedding-3-large | 1536 维度（通过 dimensions API 从 3072 减少）。约 $0.13/1M 令牌。 |
| LLM（分块/扩展） | Claude Haiku | 主题边界检测和查询扩展的最便宜模型。 |
| 后台作业 | Trigger.dev | 无服务器。嵌入回填、过期检测、孤立审核、标签一致性。 |
| 分发 | npm 包 + 编译二进制 + MCP 服务器 | OpenClaw 的库、人类的 CLI、代理的 MCP。 |

### 我们的选择及其原因

**Postgres 优于 SQLite。** 我们有 3+ 年在 Postgres 上运行的经过验证的 RAG 模式。tsvector 用于全文搜索，pgvector HNSW 用于语义搜索，pg_trgm 用于模糊 slug 匹配。将这些移植到 SQLite 将意味着从头重新实现搜索。SQLite 是未来为轻量级开源用户提供的可插拔引擎（参见 `docs/ENGINES.md`）。

**Supabase 优于自托管。** 零维护。大脑应该是 AI 代理使用的基础设施，而不是你需要管理的东西。免费层有 pgvector 但只有 500MB（不足以容纳 7K+ 带嵌入的页面，需要约 750MB）。Pro 层每月 $25 提供 8GB。v1 中没有 Docker，没有自托管 Postgres。

**完整移植优于最小可行。** 模式已经过验证。移植是机械的。交付完整的 3 层分块 + 混合搜索 + 4 层去重意味着从第一天起就有世界级的 RAG。"我们稍后添加"意味着稍后重建一切。

**库优先分发。** gbrain 是一个 npm 包。OpenClaw 作为依赖安装它（`bun add gbrain`），直接导入引擎。零开销函数调用、共享连接池、TypeScript 类型。CLI 和 MCP 服务器是同一引擎的薄包装器。

**基于触发器的 tsvector（不是生成列）。** 要在全文搜索中包含 timeline_entries 内容，tsvector 需要跨多个表。生成列无法进行跨表引用。pages + timeline_entries 上的触发器更新 search_vector。

**导入期间自动嵌入。** 无需单独的嵌入步骤。`gbrain import` 在一次传递中进行分块和嵌入。进度条显示状态。`--no-embed` 标志供想要延迟的用户使用。`embedded_at` 列支持 `gbrain embed --stale` 进行回填。

## 分发模型

```
+-------------------+     +-------------------+     +-------------------+
|   npm package     |     |  Compiled binary  |     |   MCP server      |
|   (library)       |     |  (CLI)            |     |   (stdio)         |
+-------------------+     +-------------------+     +-------------------+
|                   |     |                   |     |                   |
| bun add gbrain    |     | GitHub Releases   |     | gbrain serve      |
| import { Postgres |     | npx gbrain        |     | in mcp.json       |
|   Engine }        |     |                   |     |                   |
|                   |     |                   |     |                   |
| WHO: OpenClaw,    |     | WHO: Humans       |     | WHO: Claude Code,  |
| AlphaClaw         |     |                   |     | Cursor, etc.      |
+-------------------+     +-------------------+     +-------------------+
         |                         |                         |
         +-------------------------+-------------------------+
                                   |
                          +--------v--------+
                          |  BrainEngine    |
                          |  (pluggable     |
                          |   interface)    |
                          +-----------------+
                                   |
                     +-------------+-------------+
                     |                           |
              +------v------+            +-------v-------+
              | Postgres    |            | SQLite        |
              | Engine      |            | Engine        |
              | (v0, ships) |            | (future, see  |
              +-------------+            | ENGINES.md)   |
                                         +---------------+
```

package.json 导出：
- 库：`src/core/index.ts`（BrainEngine 接口、PostgresEngine、类型）
- CLI 二进制：`src/cli.ts`

## 首次体验

### 路径 1：OpenClaw 用户（主要）

OpenClaw 是使用 gbrain 作为其知识后端的 AI 编排器。这是最常见的安装路径。

```bash
# 1. 安装 gbrain 作为 ClawHub 技能
clawhub install gbrain

# 2. 技能在首次使用时运行引导设置：
#    - 检测 Supabase CLI 是否可用
#    - 如果是：自动配置新的 Supabase 项目
#    - 如果否：提示输入连接 URL
#    - 运行架构迁移
#    - 扫描 markdown 仓库并导入用户内容
#    - 显示实时实体/边缘提取动画
#    - 大脑就绪

# 3. 从 OpenClaw，大脑工具现在可用：
#    "Search the brain for [topic from your data]"
#    "Ingest my meeting notes from today"
#    "How many pages are in the brain?"
```

幕后，`clawhub install gbrain`：
1. 安装 `gbrain` npm 包
2. 发送 SKILL.md 文件（ingest、query、maintain、enrich、briefing、migrate）
3. 向编排器注册大脑工具
4. 在首次使用时运行 `gbrain init --supabase`（引导向导）

### 路径 2：CLI 用户（独立）

```bash
# 1. 安装
npm install -g gbrain
# 或：从 GitHub Releases 下载二进制文件

# 2. 使用 Supabase 初始化
gbrain init --supabase
# 引导向导：
#   Try 1: Supabase CLI 自动配置（npx supabase）
#   Try 2: 如果 CLI 未安装或未登录，回退：
#          "Enter your Supabase connection URL:"
#   Then: 运行架构迁移，验证 pgvector 扩展
#   Then: 验证数据库准备好导入
#   Output: "Brain ready. Run: gbrain import <your-repo>"

# 3. 导入你的数据
gbrain import /path/to/markdown/wiki/
# 进度条：7,471 files, auto-chunk, auto-embed
# 文本导入约 30 秒，嵌入约 10-15 分钟

# 4. 查询
gbrain query "what does PG say about doing things that don't scale?"
```

### 路径 3：MCP 用户（Claude Code、Cursor）

```json
// ~/.config/claude/mcp.json
{
  "mcpServers": {
    "gbrain": {
      "command": "gbrain",
      "args": ["serve"]
    }
  }
}
```

然后在 Claude Code 中："Search my brain for people who know about robotics"

### init 向导详细说明

`gbrain init --supabase` 运行以下步骤：

```
Step 1: Database Setup
  ├── Check for Supabase CLI (npx supabase --version)
  │   ├── Found + logged in → auto-create project
  │   │   ├── Create project via supabase CLI
  │   │   ├── Wait for project to be ready
  │   │   └── Extract connection string
  │   ├── Found + not logged in →
  │   │   └── Error: "Supabase CLI found but not logged in."
  │   │         Cause: "You need to authenticate first."
  │   │         Fix: "Run: npx supabase login"
  │   │         Docs: "https://supabase.com/docs/guides/cli"
  │   └── Not found → fallback to manual
  │       └── Prompt: "Enter your Supabase connection URL:"
  │
Step 2: Schema Migration
  ├── Connect to database
  ├── CREATE EXTENSION IF NOT EXISTS vector
  ├── CREATE EXTENSION IF NOT EXISTS pg_trgm
  ├── Run src/schema.sql (all tables, indexes, triggers)
  └── Verify: test insert + vector query

Step 3: Config
  ├── Write ~/.gbrain/config.json (0600 permissions)
  │   { "database_url": "...", "service_role_key": "..." }
  └── Verify connection

Step 4: Kindling Import
  ├── Import 10 bundled PG essays as demo data
  ├── Chunk + embed each essay
  ├── Show live entity/edge extraction animation:
  │   "Extracting entities... Paul Graham (person), Y Combinator (company)..."
  │   "Creating links... Paul Graham → Y Combinator (founded)..."
  └── Output: "Brain ready. 10 pages imported."

Step 5: First Query
  └── "Try: gbrain query 'what does PG say about doing things that don't scale?'"
```

每个错误都遵循风格指南：问题 + 原因 + 修复 + 文档链接。

## CLI 命令

```
gbrain init [--supabase|--url <conn>]     # 创建大脑
gbrain get <slug>                          # 读取页面
gbrain put <slug> [< file.md]             # 写入/更新页面
gbrain search <query>                      # 关键词搜索（tsvector）
gbrain query <question>                    # 混合搜索（RRF + 扩展）
gbrain ingest <file> [--type ...]         # 摄取源文档
gbrain link <from> <to> [--type <type>]   # 创建类型化链接
gbrain unlink <from> <to>                 # 删除链接
gbrain graph <slug> [--depth 5]           # 遍历链接图（递归 CTE）
gbrain backlinks <slug>                    # 入站链接
gbrain tags <slug>                         # 列出标签
gbrain tag <slug> <tag>                    # 添加标签
gbrain untag <slug> <tag>                  # 删除标签
gbrain timeline [<slug>]                   # 查看时间线
gbrain timeline-add <slug> <date> <text>  # 添加时间线条目
gbrain list [--type] [--tag] [--limit]    # 列出带过滤器
gbrain stats                               # 大脑统计
gbrain health                              # 大脑健康仪表板
gbrain import <dir> [--no-embed]          # 从 markdown 目录导入
gbrain export [--dir ./export/]           # 导出到 markdown（往返）
gbrain embed [<slug>|--all|--stale]       # 生成/刷新嵌入
gbrain serve                               # MCP 服务器（stdio）
gbrain call <tool> '<json>'               # 原始工具调用
gbrain upgrade                             # 自更新（npm、二进制、ClawHub）
gbrain version                             # 版本信息
gbrain config [get|set] <key> [value]     # 大脑配置
```

CLI 和 MCP 公开相同的操作。漂移测试断言所有操作在两个接口上的结果相同。

## 数据库架构

Postgres + pgvector 中的 9 个表：

```
+------------------+     +-------------------+     +------------------+
|     pages        |---->|  content_chunks   |     |     links        |
|------------------|     |-------------------|     |------------------|
| id (PK)          |     | id (PK)           |     | id (PK)          |
| slug (UNIQUE)    |     | page_id (FK)      |     | from_page_id(FK) |
| type             |     | chunk_index       |     | to_page_id (FK)  |
| title            |     | chunk_text        |     | link_type        |
| compiled_truth   |     | chunk_source      |     | context          |
| timeline         |     | embedding (1536)  |     +------------------+
| frontmatter(JSONB)|    | model             |
| search_vector    |     | token_count       |     +------------------+
| created_at       |     | embedded_at       |     |     tags         |
| updated_at       |     +-------------------+     |------------------|
+------------------+                                | id (PK)          |
       |                                            | page_id (FK)     |
       +-----> +--------------------+               | tag              |
       |       | timeline_entries   |               +------------------+
       |       |--------------------|
       |       | id (PK)            |               +------------------+
       |       | page_id (FK)       |               |   page_versions  |
       |       | date               |               |------------------|
       |       | source             |               | id (PK)          |
       |       | summary            |               | page_id (FK)     |
       |       | detail (markdown)  |               | compiled_truth   |
       |       +--------------------+               | frontmatter      |
       |                                            | snapshot_at      |
       +-----> +--------------------+               +------------------+
       |       |    raw_data        |
       |       |--------------------|               +------------------+
       |       | id (PK)            |               |    config        |
       |       | page_id (FK)       |               |------------------|
       |       | source             |               | key (PK)         |
       |       | data (JSONB)       |               | value            |
       |       +--------------------+               +------------------+
       |
       +-----> +--------------------+
               |   ingest_log       |
               |--------------------|
               | id (PK)            |
               | source_type        |
               | source_ref         |
               | pages_updated      |
               | summary            |
               +--------------------+
```

索引：
- `pages.slug`: UNIQUE 约束（隐式 B-tree）
- `pages.type`: B-tree
- `pages.search_vector`: GIN（全文搜索）
- `pages.frontmatter`: GIN（JSONB 查询）
- `pages.title`: GIN with pg_trgm（模糊 slug 解析）
- `content_chunks.embedding`: HNSW with cosine ops（向量搜索）
- `content_chunks.page_id`: B-tree
- `links.from_page_id`, `links.to_page_id`: B-tree
- `tags.tag`, `tags.page_id`: B-tree
- `timeline_entries.page_id`, `timeline_entries.date`: B-tree

## 搜索架构

```
Query: "when should you ignore conventional wisdom?"
           |
           v
+---------------------+
| Multi-query expansion|
| (Claude Haiku)       |
| "contrarian thinking"
| "going against the crowd"
+---------------------+
     |   |   |
     v   v   v
  [embed all 3 queries]
     |   |   |
     +---+---+
         |
    +----+----+
    |         |
    v         v
+--------+ +--------+
| Vector | | Keyword|
| Search | | Search |
| (HNSW  | | (tsv + |
| cosine)| | ts_rank)|
+--------+ +--------+
    |         |
    +----+----+
         |
         v
+------------------+
| RRF Fusion       |
| score = sum(     |
|   1/(60 + rank)) |
+------------------+
         |
         v
+------------------+
| 4-Layer Dedup    |
| 1. By source     |
| 2. Cosine > 0.85 |
| 3. Type cap 60%  |
| 4. Per-page max  |
+------------------+
         |
         v
+------------------+
| Stale alerts     |
| (compiled_truth  |
|  older than      |
|  latest timeline)|
+------------------+
         |
         v
     [Results]
```

## 分块策略

| 策略 | 输入 | 算法 | 使用场景 |
|----------|-------|-----------|-------------|
| Recursive | 任何文本 | 5 级分隔符层次结构（段落 > 行 > 句子 > 从句 > 空白）。300 词块，50 词重叠。 | 时间线（可预测格式），批量导入 |
| Semantic | 高质量文本 | 嵌入每个句子，Savitzky-Golay 滤波器用于主题边界，余弦相似度最小值。回退到递归。 | 编译真相（智能评估） |
| LLM-guided | 高价值文本 | 预分割为 128 词候选，Claude Haiku 在滑动窗口中找到主题转移。每个窗口 3 次重试。 | 通过 `--chunker llm` 明确请求 |

调度：compiled_truth 使用语义分块器。时间线使用递归分块器。通过 `--chunker` 标志或 frontmatter 中的 `chunk_strategy` 覆盖。

## 技能（胖 markdown，无代码）

每个技能都是 AI 代理（Claude Code、OpenClaw）读取并遵循的 markdown 文件。技能包含工作流、启发式和质量规则。技能逻辑不在二进制文件中。

| 技能 | 作用 |
|-------|-------------|
| `skills/ingest/SKILL.md` | 摄取会议、文档、文章。更新编译真相，追加时间线，创建链接。 |
| `skills/query/SKILL.md` | 3 层搜索（FTS + 向量 + 结构化）。用引用合成答案。 |
| `skills/maintain/SKILL.md` | 查找矛盾、过期信息、孤立页面、死链接、标签不一致。 |
| `skills/enrich/SKILL.md` | 从外部 API 丰富（Crustdata、Happenstance、Exa）。存储原始数据，提炼为编译真相。 |
| `skills/briefing/SKILL.md` | 每日简报：带背景的会议、活跃交易、开放线程。 |
| `skills/migrate/SKILL.md` | 从 Obsidian、Notion、Logseq、纯 markdown、CSV、JSON、Roam 进行通用迁移。 |

## CEO 范围扩展（v0 已接受）

1. **CLI/MCP 奇偶性与漂移测试。** 两个接口都是引擎的薄包装器。测试断言输出相同。
2. **智能 slug 解析。** 通过 pg_trgm 进行模糊匹配用于读取。写入需要精确 slug。`gbrain get "dont scale"` 解析为 `concepts/do-things-that-dont-scale`。
3. **大脑健康仪表板。** `gbrain health` 显示页面计数、嵌入覆盖率、过期页面、孤立页面、死链接。
4. **规范化时间线。** 仅 `timeline_entries` 表（无 TEXT 列）。`detail` 字段支持 markdown。
5. **页面版本控制。** `page_versions` 表存储完整快照（compiled_truth + frontmatter + links + tags）。`gbrain history`、`gbrain diff`、`gbrain revert` 命令。恢复重新分块并重新嵌入。
6. **类型化链接 + 图遍历。** `link_type` 列（knows、invested_in、works_at 等）。`gbrain graph` 使用递归 CTE，最大深度（默认 5，可通过 `--depth` 配置）。
7. **Trigger.dev 数据清理作业。** 每日嵌入回填、每周过期检测 + 孤立审核 + 标签一致性。
8. **过期警报注释。** 搜索结果标记 compiled_truth 早于最新时间线条目的页面。
9. **时间线合并摄取。** 同一事件在所有提及的实体上创建。

## 安全模型（v0）

单用户，本地仅限：
- Supabase 服务角色密钥在 `~/.gbrain/config.json`（0600 权限）
- MCP stdio 传输本质上是本地的（客户端生成 `gbrain serve` 作为子进程）
- v0 中没有多用户、没有 RLS、没有 OAuth
- 多用户路径（未来）：Supabase RLS + 每用户 API 密钥

## 升级机制

`gbrain upgrade` 检测安装方法并相应更新：

| 路径 | 方式 |
|------|-----|
| npm | `bun update gbrain`（或 npm 等效） |
| 编译二进制 | 将新二进制下载到临时目录，原子重命名交换，执行新进程 |
| ClawHub | `clawhub update gbrain` |

版本检查：将本地版本与最新 GitHub 发布标签进行比较。

## 存储和成本估算

### 存储（7,471 页约 750MB）

| 组件 | 大小 |
|-----------|------|
| 页面文本（compiled_truth + timeline） | ~150MB |
| JSONB frontmatter | ~20MB |
| tsvector + GIN 索引 | ~50MB |
| 内容块（~22K，文本） | ~80MB |
| 嵌入（22K x 1536 浮点数 x 4 字节） | ~134MB |
| HNSW 索引开销（~2x 嵌入） | ~270MB |
| 链接、标签、时间线、raw_data、版本 | ~50MB |
| **总计** | **~750MB** |

Supabase 免费层（500MB）不够。Supabase Pro（$25/月，8GB）是起点。

### 嵌入成本（初始导入约 $4-5）

| 步骤 | 成本 |
|------|------|
| 语义分块器句子嵌入（~374K 句子） | ~$1 |
| 块嵌入（~22K 块） | ~$0.30 |
| 查询扩展（每次查询约 3 次嵌入） | 可忽略 |
| **初始导入总计** | **~$4-5** |

预算替代方案：`gbrain import --chunker recursive` 跳过句子级嵌入，然后 `gbrain embed --rechunk --chunker semantic` 稍后升级。

## 无服务器操作栈

```
+------------------+     +------------------+     +------------------+
|    Supabase      |     |    Vercel         |     |   Trigger.dev    |
|  (Postgres +     |     |  (web/API,        |     |  (background     |
|   pgvector)      |     |   optional)       |     |   jobs)          |
+------------------+     +------------------+     +------------------+
| Database         |     | Future web UI     |     | Embed backfill   |
| Connection pool  |     | API endpoints     |     | Stale detection  |
| pgvector HNSW    |     | Edge functions    |     | Orphan audit     |
| tsvector FTS     |     |                   |     | Tag consistency  |
| pg_trgm fuzzy    |     |                   |     | Daily briefing   |
+------------------+     +------------------+     +------------------+
```

CLI 直接连接到 Supabase Postgres。Trigger.dev 和 Vercel 用于异步/计划工作。CLI 可以在没有它们的情况下工作。

## 验证清单

1. `gbrain import /data/brain/` 无损迁移所有 7,471 个文件
2. `gbrain export` 往返到语义相同的 markdown
3. `gbrain query "what does PG say about doing things that don't scale?"` 返回相关混合搜索结果
4. `gbrain serve` 启动 Claude Code 可连接的 MCP 服务器
5. 所有 3 个分块器使用测试 fixture 产生正确输出
6. `gbrain init --supabase` 端到端工作
7. `bun test` 通过所有测试
8. `clawhub install gbrain` 安装技能并运行引导设置
9. `bun add gbrain` + `import { PostgresEngine } from 'gbrain'` 在外部项目中工作
10. 漂移测试通过：CLI 和 MCP 产生相同结果
11. `gbrain health` 输出准确的大脑健康指标
12. 迁移技能成功导入 Obsidian vault

## 未来计划

有关可插拔引擎架构和未来后端计划，请参见 `docs/ENGINES.md`。

### v1 候选（从 v0 推迟）

- **`gbrain ask` 自然语言 CLI 别名。** 添加很简单。P1 TODO。
- **智能编译器。** 将每个事实视为一流声明，包含源跨度、实体链接、有效性窗口、置信度和矛盾状态。"发生了什么变化，为什么，什么证据会再次改变它？"来自 Codex 审查。基于编译真相模型构建。
- **通过 Trigger.dev 的主动技能。** 特定于应用的简报、会议准备。属于 OpenClaw，而不是通用大脑基础设施。
- **多用户访问。** Supabase RLS + 每用户 API 密钥。v0 是单用户。
- **SQLite 引擎。** 欢迎社区 PR。参见 `docs/SQLITE_ENGINE.md`。
- **用于自托管 Postgres 的 Docker Compose。** 欢迎社区 PR。
- **Web UI。** 可选的 Vercel 托管仪表板，用于浏览大脑页面。

### 接口抽象原则

所有操作都通过 `BrainEngine` 进行。引擎接口是契约。Postgres 特定功能（tsvector、pgvector HNSW、pg_trgm、递归 CTE）是 `PostgresEngine` 内部的实现细节。接口公开功能，而不是 SQL。

这意味着：
- SQLite 引擎可以使用 FTS5 而不是 tsvector 实现 `searchKeyword`
- SQLite 引擎可以使用 sqlite-vss 而不是 pgvector 实现 `searchVector`
- 未来的 DuckDB 引擎可以实现分析繁重的工作负载
- CLI、MCP 服务器和库使用者永远不知道底层运行的是哪个引擎

有关完整接口规范，请参见 `docs/ENGINES.md`，有关 SQLite 实现计划，请参见 `docs/SQLITE_ENGINE.md`。

## 审查历史

| 审查 | 运行次数 | 状态 | 关键发现 |
|--------|------|--------|-------------|
| /office-hours | 1 | APPROVED | Builder mode. 选择完整移植方法。 |
| /plan-ceo-review | 1 | CLEAR | 11 个提案，10 个接受，1 个推迟。SCOPE EXPANSION mode. |
| /codex review | 1 | issues_found | 24 个要点受到挑战，3 个接受（模糊 slug、恢复规范、tsvector）。 |
| /plan-eng-review | 2 | CLEAR | 3 个问题（升级路径、导入防护、init 向导），0 个关键差距。 |
| /plan-devex-review | 1 | CLEAR | DX 分数从 5/10 提高到 7/10。TTHW 从 25 分钟减少到 90 秒。Champion tier. |