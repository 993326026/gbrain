# GBrain 基础设施层

所有技能、配方和集成构建的共享基础。

## 数据管道

```
INPUT (markdown files, git repo)
  ↓
FILE RESOLUTION (local → .redirect → .supabase → error)
  ↓
MARKDOWN PARSER (gray-matter frontmatter + body)
  → compiled_truth + timeline separation
  ↓
CONTENT HASH (SHA-256 idempotency check — skip if unchanged)
  ↓
CHUNKING (3 strategies, configurable)
  ├── Recursive: 300-word chunks, 50-word overlap, 5-level delimiter hierarchy
  ├── Semantic: embed sentences, cosine similarity, Savitzky-Golay smoothing
  └── LLM-guided: Claude Haiku identifies topic shifts in 128-word candidates
  ↓
EMBEDDING (OpenAI text-embedding-3-large, 1536 dimensions)
  → batch 100, exponential backoff, non-fatal if fails
  ↓
DATABASE TRANSACTION (atomic: page + chunks + tags + version)
  ↓
SEARCH (hybrid, available immediately)
```

## 搜索架构

GBrain 使用 Reciprocal Rank Fusion (RRF) 合并向量和关键词搜索：

```
User Query
  ↓
EXPANSION (optional: Claude Haiku generates 2 alternative phrasings)
  ↓
  ├── VECTOR SEARCH (pgvector HNSW, cosine distance)
  │     → 2x limit results per query variant
  │
  └── KEYWORD SEARCH (PostgreSQL tsvector, ts_rank)
        → 2x limit results
  ↓
RRF MERGE (score = Σ(1/(60 + rank)), balances both fairly)
  ↓
4-LAYER DEDUP
  ├── Best 3 chunks per page (source dedup)
  ├── Jaccard similarity > 0.85 (text dedup)
  ├── No type exceeds 60% (diversity)
  └── Max 2 chunks per page (page cap)
  ↓
TOP N RESULTS (default 20)
```

## 关键组件

| 文件 | 用途 |
|------|---------|
| `src/core/engine.ts` | 可插拔引擎接口 (BrainEngine) |
| `src/core/postgres-engine.ts` | Postgres + pgvector 实现 |
| `src/core/import-file.ts` | importFromFile + importFromContent 管道 |
| `src/core/sync.ts` | 基于 Git 的增量变更检测 |
| `src/core/markdown.ts` | YAML frontmatter + compiled_truth/timeline 解析 |
| `src/core/embedding.ts` | OpenAI embedding 带批量、重试、退避 |
| `src/core/chunkers/recursive.ts` | 基础分块器 (300w, 5级分隔符) |
| `src/core/chunkers/semantic.ts` | 基于 embedding 的主题边界检测 |
| `src/core/chunkers/llm.ts` | Claude Haiku 引导的分块 |
| `src/core/search/hybrid.ts` | 向量 + 关键词的 RRF 合并 |
| `src/core/search/dedup.ts` | 4层结果去重 |
| `src/core/search/expansion.ts` | 通过 Claude Haiku 进行多查询扩展 |
| `src/core/storage.ts` | 可插拔存储 (S3, Supabase, local) |
| `src/core/operations.ts` | 契约优先的操作定义 (31 ops) |
| `src/schema.sql` | 完整 DDL (10 tables, RLS, tsvector, HNSW) |

## Schema 概述

Postgres 中的 10 个表：

- **pages** — slug (unique), type, title, compiled_truth, timeline, frontmatter (JSONB)
- **content_chunks** — pgvector 1536-dim embedding, chunk_source (compiled_truth|timeline)
- **links** — typed edges (knows, works_at, invested_in, founded, etc.)
- **tags** — many-to-many page tagging
- **timeline_entries** — structured events (date, source, summary, detail)
- **page_versions** — snapshot history for diff/revert
- **raw_data** — sidecar JSON from external APIs (preserves provenance)
- **files** — binary attachments in storage backend
- **ingest_log** — audit trail of import operations
- **config** — brain-level settings (version, embedding model, chunk strategy)

全文搜索使用加权 tsvector：title (A), compiled_truth (B), timeline (C)。
向量搜索使用 HNSW 索引，在 content_chunks.embedding 上使用余弦距离。

## 薄线束原则

GBrain 是确定性层。技能和配方是潜在空间层。

完整的架构理念请参见 [Thin Harness, Fat Skills](../ethos/THIN_HARNESS_FAT_SKILLS.md)。

- **GBrain CLI** = 薄线束 (same input → same output)
- **Skills** (ingest, query, maintain, enrich, briefing, migrate, setup) = 厚技能
- **Recipes** (voice-to-brain, email-to-brain) = 安装基础设施的厚技能

代理读取技能/配方并使用 GBrain 的确定性工具来完成工作。