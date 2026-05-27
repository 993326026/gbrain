# GBrain 知识运行时设计文档

**状态：** 草稿，待 CEO 审核。
**日期：** 2026-04-18.
**取代：** 早期的 "Feynman Ideas Assessment + Phase A/B" 计划。

---

## 0. Context

在对一个狭窄的双功能计划（bare-tweet citation repair + completeness score，借鉴自 Feynman）进行 CEO 审查期间，范围被重新定义。这个狭窄的计划重复了 Garry's OpenClaw 已经做的工作，并错过了真正的杠杆点：**OpenClaw 内部隐藏的定制抽象 —— resolvers、enrichment orchestration、scheduling、deterministic output —— 应该作为一等原语存在于 GBrain 中。**

北极星：*"当 Garry's OpenClaw 的 Claw 升级到此版本的 GBrain 时，它应该立即识别出 brilliance 和 completeness，并说'是时候切换到这些抽象了。'"*

这就是本文档设计所针对的测试。其他一切都是下游。

---

## 1. 四层架构

该设计包含四层抽象。每层都独立有用；它们一起构成了 Knowledge Runtime。

```
  ┌───────────────────────────────────────────────────────────────────┐
  │                   KNOWLEDGE RUNTIME (new)                         │
  ├───────────────────────────────────────────────────────────────────┤
  │  Layer 4: Deterministic Output Builder                            │
  │     BrainWriter · Scaffolds · Back-link enforcer · Slug registry  │
  │     Rule: LLM picks WHAT to write. Code guarantees WHERE and HOW. │
  ├───────────────────────────────────────────────────────────────────┤
  │  Layer 3: Scheduler                                               │
  │     ScheduledResolver · TZ-aware quiet hours (enforced) ·         │
  │     Auto-stagger · Durable state · Retry/circuit-break            │
  ├───────────────────────────────────────────────────────────────────┤
  │  Layer 2: Enrichment Orchestrator                                 │
  │     Trigger convergence · Tier routing · Budget · Cascade ·       │
  │     Evidence-weighted completeness · Fail-safe transactions       │
  ├───────────────────────────────────────────────────────────────────┤
  │  Layer 1: Resolver SDK                                            │
  │     Resolver<I,O> interface · Registry · Factory · Plugin recipes │
  │     Ported reference impls: X-API, Perplexity, Mistral, brain     │
  └───────────────────────────────────────────────────────────────────┘
          │                                                │
          ▼                                                ▼
     REUSES (polished primitives already in GBrain)  REPLACES (ad-hoc code)
     FailImproveLoop · backoff · storage factory ·   enrichment-service ·
     check-resolvable · operations validators ·      embedding · transcription ·
     engine interface · publish · backlinks          2 recipe formats
```

---

## 2. 为什么是这个顺序（L1 → L4）

每一层都依赖于下一层。**L1 必须首先落地，否则其余层会泄漏抽象。**

- **L1（Resolvers）** 是基础。没有统一的查找接口，每个 orchestrator + writer 都有定制的调用者。
- **L2（Orchestrator）** 使用 L1 来获取数据；没有 L1，它仍然是临时的。
- **L3（Scheduler）** 定期运行 L2；没有 L2，它调度的是无结构的内容。
- **L4（Output Builder）** 是每一层最终写入的地方；没有它，我们有 14 个调用站点使用手工的引用规范进行 `fs.writeFile`。

早期实现可以先发布 L1 + L4（两个"最纯粹"的层），产生最直接的完整性影响，然后添加 L2 + L3。但最终状态必须包含所有四层。

---

## 3. Layer 1 — Resolver SDK

### 3.1 当前存在的问题

Garry's OpenClaw 有 **69 种不同的外部查找模式**，涵盖 X API（14 种形状）、Perplexity、Mistral OCR、Gmail、Calendar、Slack、GitHub、YouTube、Diarize.io、YC tools、OSINT 收集器和大脑本地查找。每个都是 `scripts/` 下的定制脚本，有自己的错误处理、重试逻辑和输出形状。GBrain 有 3 个临时包装器（`embedding.ts`、`transcription.ts`、`enrichment-service.ts`），它们不共享接口。

常见后果：
- 没有统一的重试/退避策略（有些脚本重试，大多数不重试）
- 没有成本跟踪（当调用返回无实质结果时，Perplexity 费用被静默消耗）
- 没有置信度/来源传播（调用者无法判断答案是验证的还是推断的）
- 用户不能在不 fork GBrain 的情况下添加 resolver

### 3.2 接口

```typescript
// src/core/resolvers/interface.ts

export type ResolverCost = 'free' | 'rate-limited' | 'paid';

export interface ResolverRequest<I> {
  input: I;
  context: ResolverContext;
  timeoutMs?: number;
}

export interface ResolverResult<O> {
  value: O;
  confidence: number;      // 0.0–1.0; 1.0 = deterministic from ground-truth API
  source: string;          // e.g. "x-api-v2", "perplexity-sonar", "brain-local"
  fetchedAt: Date;
  costEstimate?: number;   // dollars; 0 if free
  raw?: unknown;           // for sidecar preservation via put_raw_data
}

export interface Resolver<I, O> {
  readonly id: string;           // stable, slug-like: "x_handle_to_tweet"
  readonly cost: ResolverCost;
  readonly backend: string;      // "x-api-v2", "perplexity", "brain-local"
  readonly inputSchema: JSONSchema;
  readonly outputSchema: JSONSchema;

  available(ctx: ResolverContext): Promise<boolean>;
  resolve(req: ResolverRequest<I>): Promise<ResolverResult<O>>;
}
```

### 3.3 上下文

```typescript
export interface ResolverContext {
  engine: BrainEngine;
  storage: StorageBackend;
  config: GBrainConfig;
  logger: Logger;
  metrics: MetricsRecorder;
  budget: BudgetLedger;       // hard spend caps, queried pre-resolve
  requestId: string;
  remote: boolean;            // trust boundary — untrusted callers get stricter validation
  deadline?: Date;
}
```

### 3.4 Registry + Factory（镜像 `src/core/storage.ts`）

```typescript
// src/core/resolvers/registry.ts
export class ResolverRegistry {
  register<I, O>(r: Resolver<I, O>): void;
  get(id: string): Resolver<unknown, unknown>;
  list(filter?: { cost?: ResolverCost; backend?: string }): Resolver[];
  async resolve<I, O>(id: string, input: I, ctx: ResolverContext): Promise<ResolverResult<O>>;
}

// src/core/resolvers/factory.ts (dynamic import like engine-factory)
export async function createResolver(
  type: 'x-api' | 'perplexity' | 'mistral-ocr' | 'brain-local' | 'plugin',
  config: ResolverConfig,
): Promise<Resolver>;
```

### 3.5 插件格式（统一 `recipes/` + `data-research` 格式）

插件是 YAML + JS 模块，通过文件系统扫描 `~/.gbrain/resolvers/` 和 `recipes/` 发现。

```yaml
# Example: resolvers/x-api/handle-to-tweet.yaml
id: x_handle_to_tweet
version: 1
category: lookup
cost: rate-limited
backend: x-api-v2
module: ./handle-to-tweet.ts
input_schema:
  type: object
  properties:
    handle:   { type: string, pattern: "^[A-Za-z0-9_]{1,15}$" }
    keywords: { type: string }
  required: [handle]
output_schema:
  type: object
  properties:
    url:        { type: string, format: uri }
    tweet_id:   { type: string }
    text:       { type: string }
    created_at: { type: string, format: date-time }
requires:
  env: [X_API_BEARER_TOKEN]
health_check:
  kind: http
  url: https://api.twitter.com/2/tweets/1
  expect: { status: [200, 401] }   # 401 = auth failure but endpoint reachable
tests:
  - input:  { handle: "garrytan" }
    expect: { url: { pattern: "^https://x\\.com/garrytan/status/\\d+$" } }
```

信任标记遵循现有的 `src/commands/integrations.ts` 模式：只有包捆绑的 resolvers 是 `embedded=true`，可以运行任意命令；用户提供的 resolvers 仅限于 `http` 和验证的 schemas。

### 3.6 用 `FailImproveLoop` 包装每个 resolver

现有的 `src/core/fail-improve.ts` 是确定性优先/LLM 回退模式。每个 resolver 自动被包装：如果确定性路径（例如 X API）返回有效结果，则使用它；如果失败，可选回退到基于 LLM 的 resolver；记录两条路径用于未来模式分析和自动测试生成。

### 3.7 要发布的参考实现

OpenClaw 调查盘点了 69 种 resolver 形状。全部发布是错误的（范围过大）；零发布是范围不足的。dogfood 集：

| # | Resolver | 用途 | 使用者 |
|---|---|---|---|
| 1 | `x_handle_to_tweet` | Bare-tweet citation repair (原始 Phase A) | `gbrain integrity` |
| 2 | `url_reachable` | 死链接检测 | `gbrain integrity` |
| 3 | `brain_slug_lookup` | Name/email → slug (包装现有的 `resolveSlugs`) | Output Builder |
| 4 | `openai_embedding` | 将 `src/core/embedding.ts` 重构为 Resolver | Import pipeline |
| 5 | `perplexity_query` | Query → synthesis + citations | Enrichment Orchestrator |
| 6 | `text_to_entities` | LLM entity extraction (structured JSON) | Enrichment Orchestrator |

剩余的 63 种 OpenClaw 模式根据用户需求逐步移植。每个移植是 `recipes/` 或 `~/.gbrain/resolvers/` 下的新 YAML + 模块，无需框架更改。

---

## 4. Layer 2 — Enrichment Orchestrator

### 4.1 当前存在的问题

Garry's OpenClaw 的 enrichment **在数据层很完善，在控制层很简陋**：

- **Completeness = "length > 500 chars + no `needs-enrichment` tag"** (`lib/enrich.mjs:351-355`)。很幼稚。一个充满重复 Perplexity 摘要的丰富页面（见 `brain/people/0interestrates.md` — 38 个重复块）通过此检查。
- **30-day auto-re-enrichment** 永远运行。没有"完成"状态。2023 年见过一次的人仍然每月被重新研究。
- **Cascade is convention-only.** Person→company stubs 自动创建；company→investors、company→employees 遍历有文档记录但从未实现。
- **No hard budget cap.** 成本按批次估算，从未跨批次或每天强制执行。
- **Failure is silent.** 错误的 Perplexity 响应记录并继续；部分写入可能使页面留下时间线条目但没有原始数据 sidecar。

### 4.2 Orchestrator

```typescript
// src/core/enrichment/orchestrator.ts

export interface EnrichmentRequest {
  entitySlug: string;
  trigger: 'mention' | 'stub-creation' | 'cron-sweep' | 'manual' | 'cascade';
  tier?: 1 | 2 | 3;                // optional override; auto-computed if absent
  cascadeDepth?: number;           // 0 = no cascade; default 1
}

export interface EnrichmentResult {
  entitySlug: string;
  completenessBefore: number;
  completenessAfter: number;
  resolversUsed: string[];         // e.g. ["perplexity_query", "x_handle_to_tweet"]
  costSpent: number;
  writtenTo: string[];             // page paths touched, for transaction audit
  cascadedTo: string[];            // related entities enriched
  status: 'enriched' | 'skipped' | 'failed' | 'budget-exhausted';
  reason?: string;
}

export class EnrichmentOrchestrator {
  constructor(
    private registry: ResolverRegistry,
    private writer: BrainWriter,
    private budget: BudgetLedger,
    private scorer: CompletenessScorer,
    private graph: EntityGraph,
  ) {}

  async enrich(req: EnrichmentRequest): Promise<EnrichmentResult>;
  async enrichBatch(reqs: EnrichmentRequest[]): Promise<EnrichmentResult[]>;
}
```

### 4.3 证据加权的 completeness（取代长度启发式）

Completeness 是每个实体类型的评分标准，写入时存储在前言中，并按需重新计算。

```typescript
// src/core/enrichment/completeness.ts
export interface CompletenessRubric<Page> {
  entityType: PageType;
  dimensions: {
    name: string;
    weight: number;                // sum must = 1.0
    check: (page: Page) => number; // 0.0–1.0
  }[];
}

// Example rubric for persons:
//   - has_role_and_company   0.20
//   - has_source_urls        0.20  (≥1 URL with resolver-verified reachability)
//   - has_timeline_entries   0.15  (≥1)
//   - has_citations          0.15  (every claim has [Source: ...])
//   - has_backlinks          0.10  (every linked page links back)
//   - recency_score          0.10  (last_verified within 90 days)
//   - non_redundancy         0.10  (no repeated blocks; distinct-lines/total-lines > 0.8)
```

**关键属性：** `non_redundancy` + `recency_score` 明确消除了审计中观察到的两个大脑病态（Wilco 风格的重复块；没有 `last_verified` 的陈旧页面）。

`completeness` 字段以前言形式存在，值为 `0.0–1.0`。可以通过 `list_pages(where: completeness < 0.5)` 查询。

### 4.4 带硬预算的 Tier 路由

二维路由：**重要性**（来自 person-score 的 tier 1/2/3）× **预算状态**。

```typescript
// src/core/enrichment/tiers.ts
export const TIER_CONFIG = {
  1: { models: ['opus', 'sonar-deep'], maxCostUsd: 0.10, cascadeDepth: 2 },
  2: { models: ['sonar'],              maxCostUsd: 0.02, cascadeDepth: 1 },
  3: { models: ['sonar'],              maxCostUsd: 0.005, cascadeDepth: 0 },
};

// src/core/enrichment/budget.ts
export class BudgetLedger {
  // Hard caps. Queryable pre-resolve.
  dailyCapUsd: number;
  perEntityCapUsd: number;
  perResolverCapUsd: Map<string, number>;

  async reserve(resolverId: string, estimateUsd: number): Promise<Reservation | 'exhausted'>;
  async commit(reservation: Reservation, actualUsd: number): Promise<void>;
  async rollback(reservation: Reservation): Promise<void>;
  async state(): Promise<{ spent: number; remaining: number; perResolver: Record<string, number> }>;
}
```

**属性：** 如果达到每日上限，`orchestrator.enrich()` 立即返回 `status: 'budget-exhausted'`。没有静默超支。断路器在用户配置的时区午夜重置。

### 4.5 Cascade（实体图遍历）

```typescript
// src/core/enrichment/cascade.ts
export class EntityGraph {
  // Deterministic, no LLM. Uses engine.getLinks() + engine.getBacklinks().
  async neighbors(slug: string, depth: number): Promise<string[]>;
  async cascadeFrom(trigger: string, depth: number): Promise<EnrichmentRequest[]>;
}
```

如果人员 X 被丰富并获得新的 `company: Acme` 字段，cascade 检查：`companies/acme` 是否存在？如果不存在，创建 stub + 在 tier 2 入队。`companies/acme` 是否链接回 X？如果没有，写入反向链接。**Iron Law 是机器强制执行的，不是技能强制执行的。**

### 4.6 故障安全事务

每次 enrichment 都包装在 BrainWriter 事务中（Layer 4）。部分写入回滚。没有时间线条目无原始 sidecar 这样的不对称状态。

```typescript
await writer.transaction(async (tx) => {
  const research = await registry.resolve('perplexity_query', {...}, ctx);
  await tx.appendTimeline(slug, {...});
  await tx.putRawData(slug, 'perplexity', research.raw);
  await tx.setFrontmatterField(slug, 'completeness', score);
  // All-or-nothing commit on exit.
});
```

---

## 5. Layer 3 — Scheduler

### 5.1 当前存在的问题

Garry's OpenClaw 的 cron 是**外部驱动的 JSON**（`cron/jobs.json`），约 30 个作业手动错开在不同分钟。GBrain **没有原生调度** —— `src/commands/autopilot.ts` 是单个守护进程循环，`docs/guides/cron-schedule.md` 是架构指导，不是代码。

在 Garry's OpenClaw 的实际状态中观察到的失败：
- `X OAuth2 Token Refresh`: 11 次连续超时（关键路径静默失败）
- `flight-tracker daily scan`: 5 次连续超时
- `morning-briefing`: 4 次连续超时
- 安静时间在技能运行时检查，因此忘记检查的技能会在凌晨 3 点发送 DM。
- 错开是手动约定；配置编辑后没有防止两个作业冲突的保护。

### 5.2 ScheduledResolver 接口

```typescript
// src/core/scheduling/scheduler.ts
export interface Schedule {
  kind: 'cron' | 'interval';
  expr?: string;                    // cron string
  intervalMs?: number;
  tz: string;                       // IANA: "America/Los_Angeles"
  quietHours?: {
    startHour: number;              // 22 = 10 PM local
    endHour: number;                // 7 = 7 AM local
    policy: 'skip' | 'defer' | 'silent-run';
  };
  staggerKey?: string;              // jobs with same key auto-offset
  maxConcurrent?: number;           // global concurrency cap
  maxDurationMs?: number;           // timeout
}

export interface ScheduledResolver extends Resolver<void, ScheduledResult> {
  schedule: Schedule;
  retryPolicy: { maxRetries: number; backoffMs: number };
  circuitBreaker: { failureThreshold: number; cooldownMs: number };
  state: DurableState;              // watermark, content-hash, idempotency key
}
```

### 5.3 强制执行 vs 约定（与 Garry's OpenClaw 的关键差异）

| 关注点 | Garry's OpenClaw 现状 | Knowledge Runtime |
|---|---|---|
| Quiet hours | 在每个技能内部检查（基于信任） | 在调度器强制执行，技能无法覆盖 |
| Staggering | `jobs.json` 中手动分钟偏移 | 调度器通过哈希 staggerKey 分配槽位 |
| Concurrency | `MAX_BATCH_PROCESSES=2` 在 backoff 中，cron 忽略 | 调度器中的全局信号量 |
| Timeout | JSON 中的每作业字符串，不总是被尊重 | 通过 `AbortController` 强制执行，超时引发 `TimeoutError` 被 orchestrator 捕获 |
| Retry | cron 级别无 | 带指数退避的 `retryPolicy` |
| Silent failure | "11 consecutive timeouts" 未被注意 | 断路器在阈值打开 → 升级到用户 |
| Idempotency | 每作业状态文件，无框架 | `DurableState` 原语：watermark/ID/content-hash |

### 5.4 原生引擎 + OS cron 适配器

调度器以以下两种方式之一运行：
1. **嵌入式**（`gbrain autopilot` 默认）：守护进程内的原生事件循环。一个进程，多个 ScheduledResolvers。
2. **OS 驱动**（用于 Railway/launchd/systemd）：`gbrain schedule run <id>` 由 OS cron 调用，调度器状态持久化，因此跨调用去重仍然有效。

两种模式共享相同的 `Schedule` 配置 + 状态。

### 5.5 可观测性

每个计划运行发出结构化事件：`started`、`skipped-quiet-hours`、`deferred-to-active-hours`、`failed-retrying`、`circuit-opened`、`completed`。事件发送到：
- `~/.gbrain/scheduler/events.jsonl`（本地，始终）
- `engine.logIngest`（大脑数据库中的审计跟踪）
- 可选 webhook（用户的 Slack/Telegram）

`gbrain doctor` 读取事件日志并报告：当前断路器状态、任何连续失败 > 3 次的 resolver、任何在 3× 间隔内未触发的 resolver（像 Garry's OpenClaw 的 `freshness-check.mjs` 但内置的新鲜度 SLA）。

---

## 6. Layer 4 — Deterministic Output Builder

### 6.1 反幻觉不变量

**Iron Law: LLM picks WHAT. Code guarantees WHERE and HOW.**

Garry's OpenClaw 现有的 `lib/enrich.mjs:buildTweetEntry` 接近这个 —— tweet URL 由 X API 返回的 `tweet.id` 构建，从不来自 LLM 记忆。但是：

- 过去的事件：*"Sub-agent test #2 FAILED — hallucinated 'Philip Leung' entity links across all daily files. LLM rewriting of daily files is too error-prone."*（Garry's OpenClaw memory log, 2026-04-13.）
- 反向链接依赖于在各处调用 `appendTimeline`；跳过是静默的。
- Slug 冲突未检查（`slugify` 上无冲突检测）。
- 引用格式每周事后 lint，不是写入前强制执行。

### 6.2 BrainWriter

```typescript
// src/core/output/writer.ts
export class BrainWriter {
  constructor(
    private engine: BrainEngine,
    private slugRegistry: SlugRegistry,
    private scaffolder: Scaffolder,
  ) {}

  async transaction<T>(fn: (tx: WriteTx) => Promise<T>): Promise<T>;
}

export interface WriteTx {
  // High-level typed operations; never raw string writes.
  createEntity(input: EntityInput): Promise<string>;          // returns slug, conflict-checked
  appendTimeline(slug: string, entry: TimelineInput): Promise<void>;
  setCompiledTruth(slug: string, body: CompiledTruthInput): Promise<void>;
  setFrontmatterField(slug: string, key: string, value: unknown): Promise<void>;
  putRawData(slug: string, source: string, data: object): Promise<void>;
  addLink(from: string, to: string, context: string): Promise<void>;  // auto-creates reverse back-link

  // Validators (called implicitly on commit)
  validate(): Promise<ValidationReport>;
}
```

### 6.3 Scaffolder — 确定性链接 + 引用构造

每个用户可见的 URL/链接/引用都由代码从 resolver 输出构建，而不是从 LLM 文本。

```typescript
// src/core/output/scaffold.ts
export class Scaffolder {
  tweetCitation(handle: string, tweetId: string, dateISO: string): string {
    // "[Source: [X/garrytan, 2026-04-18](https://x.com/garrytan/status/123456)]"
  }
  emailCitation(account: string, messageId: string, subject: string): string {
    // deterministic Gmail URL per OpenClaw pattern
  }
  sourceCitation(resolverResult: ResolverResult<unknown>): string {
    // pulls .source, .fetchedAt, .raw from the result
  }
  entityLink(slug: string): string {
    // slugRegistry checks existence; returns resolvable wikilink
  }
}
```

### 6.4 SlugRegistry — 冲突检测

```typescript
// src/core/output/slug-registry.ts
export class SlugRegistry {
  async create(desiredSlug: string, displayName: string, type: PageType): Promise<CreatedSlug>;
  // Throws SlugCollision if another entity already occupies desiredSlug and isn't
  // confirmed as the same person (via email / x_handle / disambiguator).
  // Auto-resolves near-collisions by appending disambiguator.

  async confirmSame(slugA: string, slugB: string, confidence: number): Promise<void>;
  async merge(canonical: string, duplicate: string): Promise<void>;
}
```

### 6.5 写入前验证器（完整性失败关闭）

在提交前的 `WriteTx.validate()` 上：

1. **引用验证器。** `compiled_truth` 中的每个事实句子必须在 N 行内有内联 `[Source: ...]`。不兼容的段落被标记。可配置：严格模式拒绝事务，lint 模式警告。
2. **链接验证器。** 每个 `[text](path)` 必须指向存在的页面 OR Scaffolder 构建的 URL（因此保证有效）。没有原始 LLM 组合的 URL。
3. **反向链接验证器。** 每个出站链接必须在同一事务中写入反向链接。
4. **Triple-HR 验证器。** Compiled truth / timeline 分割在 schema 级别强制执行。

**失败关闭**：默认是严格模式。放宽需要显式 `writer.transaction({ strictMode: false }, ...)` 并向 ingest log 记录警告。

### 6.6 LLM 输出清理

任何发往大脑页面的 LLM 输出首先通过 JSON-Schema 验证的解析器。没有自由格式的 markdown 写入磁盘。

- 实体提取：JSON 数组 `{ name, type, context }` 按照现有的 `extractEntities` 模式 —— 严格验证。
- Compiled-truth synthesis：LLM 发出结构化的 `{ sections: [{heading, paragraphs: [{text, sources: [...]}]}]}`，scaffolder 渲染为 markdown。
- Timeline entries：LLM 发出 `{ date, summary, detail, sources }`，scaffolder 渲染。

LLM 永远看不到文件路径，从不写入文件，从不发出完成的 markdown。

---

## 7. 与现有 GBrain 的集成

### 7.1 重用（已完善）

| 现有 | 使用者 | 变更 |
|---|---|---|
| `src/core/fail-improve.ts` (9/10) | 在 L1 中包装每个 Resolver | 无；成为默认包装器 |
| `src/core/backoff.ts` (9/10) | ResolverContext.backoff | 无 |
| `src/core/storage.ts` (9/10) | Resolver factory 模式模板 | 无；作为模式参考 |
| `src/core/check-resolvable.ts` (9/10) | 扩展以验证 Resolver 插件 | 添加 `checkResolvers()` 模式 |
| `src/commands/publish.ts` (9/10) | 在底层使用 BrainWriter | 次要：通过 L4 路由 |
| `src/commands/backlinks.ts` (8/10) | 折叠到 L4 验证器 | 保留作为 CLI 面对的 lint 入口点 |
| `src/core/operations.ts` validators | 在 ResolverContext 信任强制执行中重用 | 无 |
| `src/core/engine.ts` BrainEngine (35 methods) | ResolverContext.engine | 扩展 `getResolverRegistry()` |

### 7.2 替换（当前临时）

| 现有 | 替换为 |
|---|---|
| `src/core/enrichment-service.ts` (5/10) | `src/core/enrichment/orchestrator.ts` (L2) |
| `src/core/embedding.ts` (monolithic) | `src/core/resolvers/builtin/embedding/openai.ts` |
| `src/core/transcription.ts` (monolithic) | `src/core/resolvers/builtin/transcription/{groq,openai}.ts` |
| `src/commands/integrations.ts` recipe format | 统一的 Resolver 插件格式（§3.5） |
| `src/core/data-research.ts` recipe format | 相同的统一格式 |
| `src/commands/autopilot.ts` hard-coded daemon loop | 包装一组 ScheduledResolvers |

### 7.3 扩展

- `src/core/engine.ts`: 添加 `getResolverRegistry()`、`getWriter()`、`getScheduler()`。引擎成为运行时的根容器。
- `src/core/operations.ts`: `OperationContext` 继承自 `ResolverContext`（或反之）。信任标志统一。
- `src/core/types.ts`: 向 `Page` 添加 `completeness: number`，为来源添加 `sourcedBy: string[]`。

---

## 8. 迁移路径（分阶段，可发布）

每个阶段独立发布，通过完整 E2E，有功能标志，可逆。没有大爆炸。

### Phase 0 — Foundation（人力：~1 周 / CC：~4 小时）
- 定义 `Resolver<I,O>`、`ResolverContext`、`ResolverRegistry`、`ResolverResult`（§3.2–3.4）。
- 添加 `src/core/resolvers/index.ts` 接线 + 注册表测试（register/get/list）。
- 无行为变更；作为带功能标志的 `v0.11.0-alpha` 发布。

### Phase 1 — 三个参考 resolvers（人力：~1 周 / CC：~4 小时）
- 移植 `src/core/embedding.ts` → `resolvers/builtin/embedding/openai.ts`。
- 实现 `resolvers/builtin/brain-local/slug-lookup.ts`（包装 `engine.resolveSlugs`）。
- 实现 `resolvers/builtin/url-reachable.ts`（HEAD-check）。
- 证明接口：旧调用者切换到 `registry.resolve('openai_embedding', ...)`。

### Phase 2 — BrainWriter + Slug Registry（人力：~1.5 周 / CC：~6 小时）
- L4 core：`BrainWriter.transaction`、`Scaffolder`、`SlugRegistry` 带冲突检测。
- 写入前验证器：引用、链接、反向链接、triple-HR。
- 迁移 `src/commands/publish.ts` + `src/commands/backlinks.ts` 以通过 BrainWriter 路由。
- **现在** Garry's OpenClaw 的 "Philip Leung" 幻觉在结构上是不可能的 —— LLM 输出在到达 Scaffolder 之前通过 JSON-Schema 验证器。

### Phase 3 — `gbrain integrity` 命令（人力：~0.5 周 / CC：~2 小时）
- 在新基础之上发布最初范围的面向用户的功能。
- 使用 Resolver SDK：`x_handle_to_tweet` + `url_reachable`。
- 使用 BrainWriter：所有自动修复都通过验证写入。
- `--auto --confidence 0.8` 模式如用户在 cherry-pick #1 中批准。
- **用户可见的值在 Phase 3 发布，不是 Phase 7。**

### Phase 4 — Enrichment Orchestrator（人力：~2 周 / CC：~8 小时）
- L2 core：`EnrichmentOrchestrator`、`BudgetLedger`、`CompletenessScorer`、`EntityGraph.cascadeFrom`。
- 迁移 `src/core/enrichment-service.ts` 调用者（之后弃用旧文件）。
- 每次写入时在前言中写入 Completeness score（dogfooding cascades）。

### Phase 5 — Scheduler（人力：~2 周 / CC：~8 小时）
- L3 core：`Scheduler`、`ScheduledResolver`、`DurableState`、断路器、安静时间强制执行器。
- 迁移 `src/commands/autopilot.ts` 到 ScheduledResolver 集。
- 发布 `gbrain schedule list|run|pause|tail` CLI 用于可观测性。

### Phase 6 — Port 5–8 OpenClaw resolvers（人力：~1.5 周 / CC：~6 小时）
- `perplexity_query`、`text_to_entities`、`mistral_ocr_pdf`、`x_search_all`、`x_user_to_tweets`、`gmail_query_to_threads`、`calendar_date_to_events`。
- 每个作为 `resolvers/builtin/` 下的 YAML + TS 模块发布 —— **插件格式的证明。**

### Phase 7 — OpenClaw Adoption Integration（人力：~1 周 / CC：~4 小时）
- 编写 `docs/openclaw/ADOPTION.md`，展示您的 OpenClaw 如何用对 `gbrain registry.resolve(...)` 的调用来替换其 69 个定制脚本。
- 发布 `gbrain claw-bridge` 子命令，将 Garry's OpenClaw 当前的脚本调用代理到 resolver 注册表 —— 零编辑采用路径。
- **这是北极星的测试。** 如果您的 OpenClaw 可以建立一行 shim 并删除 `scripts/x-api-client.mjs`，抽象就成功了。

总计：人力：~10 周 / CC：~42 小时 / 单人实现日历时间：~3–4 周。

---

## 9. 关键文件

### 新目录 / 文件

```
src/core/
  runtime/
    index.ts                       # RuntimeContext (engine, storage, config, logger, metrics, budget)
    registry.ts                    # ResolverRegistry
    factory.ts                     # createResolver()
  resolvers/
    interface.ts                   # Resolver<I, O>
    fail-improve-wrapper.ts        # auto-wraps every resolver in FailImproveLoop
    builtin/
      x-api/
        handle-to-tweet.ts
        handle-to-tweet.yaml
      perplexity/
        query.ts
        query.yaml
      brain-local/
        slug-lookup.ts
        url-reachable.ts
      embedding/
        openai.ts                  # refactored from src/core/embedding.ts
      transcription/
        groq.ts
        openai.ts
  enrichment/
    orchestrator.ts                # EnrichmentOrchestrator
    tiers.ts                       # TIER_CONFIG
    budget.ts                      # BudgetLedger
    completeness.ts                # CompletenessScorer + per-type rubrics
    cascade.ts                     # EntityGraph
  scheduling/
    scheduler.ts                   # Scheduler + ScheduledResolver
    schedule.ts                    # Schedule type, cron expr parser
    state.ts                       # DurableState primitives
    quiet-hours.ts                 # TZ-aware enforcement
    stagger.ts                     # deterministic slot assignment
  output/
    writer.ts                      # BrainWriter
    scaffold.ts                    # Scaffolder (typed URL builders)
    slug-registry.ts               # SlugRegistry (conflict detection)
    validators/
      citation.ts
      link.ts
      back-link.ts
      triple-hr.ts

src/commands/
  integrity.ts                     # ships in Phase 3, replaces Feynman Phase A/B
  schedule.ts                      # gbrain schedule list|run|pause|tail (Phase 5)

docs/openclaw/
  ADOPTION.md                      # written in Phase 7
```

### 替换 / 删除
- `src/core/enrichment-service.ts` — 折叠到 `enrichment/orchestrator.ts`
- `src/core/embedding.ts` — 移动到 `resolvers/builtin/embedding/openai.ts`
- `src/core/transcription.ts` — 移动到 `resolvers/builtin/transcription/`

### 扩展
- `src/core/engine.ts` — 添加 `getResolverRegistry()`、`getWriter()`、`getScheduler()`
- `src/core/operations.ts` — 与 ResolverContext 统一；每个操作验证器可被 resolvers 重用
- `src/core/types.ts` — 添加 `completeness: number`、`sourcedBy: string[]`、`lastVerified: Date`

---

## 10. 测试策略

### 契约测试
每个 Resolver 实现针对接口规范测试。表驱动：对 `openai_embedding`、`x_handle_to_tweet` 等运行相同的套件。确保插件作者不能发布损坏的 resolvers。

### 属性测试
- **幂等性：** 使用相同状态运行 ScheduledResolver 两次产生相同输出，不会重复写入。
- **原子性：** 中途抛出的 BrainWriter 事务使大脑逐位等同于事务前。
- **确定性脚手架：** 给定相同的 resolver 输出，Scaffolder 生成字节相同的引用/链接。

### 集成测试
- `EnrichmentOrchestrator` 端到端针对 PGLite（内存中，无 API 密钥）与模拟的 resolver 注册表。
- `Scheduler` 带假时钟 + 安静时间场景。
- 验证器失败时的 BrainWriter 事务回滚。

### 混沌测试
- 丰富过程中杀死进程；下次运行必须干净地恢复。
- 模拟事务中途 API 超时；事务必须完全回滚。
- 损坏的状态文件；调度器必须升级，而不是静默跳过。

### 与 Garry's OpenClaw 行为的回归测试
对于我们移植的每个 OpenClaw 模式（例如 X-handle → tweet URL），回归测试证明新 resolver 在来自大脑审计的真实世界输入上产生相同的答案。这是"您的 OpenClaw 会采用"的证明。

---

## 11. 开放问题（标记为 CEO 重新审查）

1. **范围形状。** 这是正确的四层分解，还是某些层最好留给 OpenClaw（例如 Scheduling 位于 GBrain 之上，而不是其中）？
2. **Phase 3 用户价值突破。** Phase 3（用户可见的 `gbrain integrity`）是否足够早发布，还是我们需要更小的 MVP？
3. **LLM-as-resolver。** `text_to_entities` 应该是 Resolver，还是这模糊了不变量依赖的"代码 vs LLM"界限？
4. **插件格式。** YAML + TS 模块（§3.5）vs 纯 TS 模块带装饰器风格元数据。后者更类型安全；前者更易发现。
5. **Cross-resolver transactions。** 我们在 L2 层支持"原子 fetch-from-Perplexity + write-to-brain"吗？当前设计说是；实现很棘手（Perplexity 调用不可回滚）。
6. **OpenClaw bridge scope。** Phase 7 `gbrain claw-bridge` — 这值得自己的阶段，还是采用应该仅文档化？
7. **Completeness rubric coverage。** 我们是否预先为所有 9 个 PageTypes 定义 rubrics，还是先发布人员/公司/会议并逐步扩展？
8. **预算配置 UX。** 硬每日上限很严格；我们是否也应该公开软上限警告模式，以及如何设置上限（环境变量？配置文件？首次使用时提示？）
9. **向后兼容。** `src/commands/publish.ts` 和 `src/commands/backlinks.ts` 已经干净运行了几周。通过 BrainWriter 重构存在迁移风险。可接受？
10. **现有 TODOS 对齐。** `TODOS.md` 有 P0 "Runtime MCP access control" 和 P2 安全强化。新的 RuntimeContext.remote 标志与两者交互 —— 我们将 MCP 访问控制折叠到 Phase 0 还是保持分离？

---

## 12. 验证（"您的 OpenClaw 会采用"测试）

设计成功当且仅当：

- [ ] 用户可以通过在 `~/.gbrain/resolvers/` 中放置 YAML + TS 模块来添加新 resolver，无需编辑 GBrain 源代码。
- [ ] 您的 OpenClaw 可以删除 `scripts/x-api-client.mjs` 并用 1 行 `await registry.resolve('x_handle_to_tweet', ...)` 替换所有调用者。
- [ ] 大脑页面不能用裸 tweet 引用、缺失的反向链接或未验证的 URL 写入（验证器在提交前捕获）。
- [ ] 对真实大脑运行 `gbrain integrity --auto --confidence 0.8` 修复 ≥1,000 个已知的 1,424 个裸 tweet 引用，无需人工审查。
- [ ] 完整的 E2E 测试套件在 PGLite + Postgres 引擎上通过。
- [ ] Knowledge Runtime 跨越 7 个阶段发布，每个阶段可单独发布且可逆。