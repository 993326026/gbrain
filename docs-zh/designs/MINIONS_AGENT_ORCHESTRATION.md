---
status: ACTIVE
---
# CEO 计划：Minions 作为通用代理编排协议
由 /plan-ceo-review 于 2026-04-15 生成
分支：garrytan/minions-jobs | 模式：SCOPE EXPANSION
仓库：garrytan/gbrain

## 愿景

### 10x 检查
与其说"GBrain 有队列，OpenClaw 使用它"，不如让 Minions 成为通用代理
编排协议。任何平台（OpenClaw、Hermes、Claude Code、Codex、自定义
脚本）通过相同的 Postgres-native 协议提交、监控、引导和组合代理。GBrain 就是代理控制平面。

### 柏拉图式理想（理想北极星，不在 v1 范围内）
打开终端，输入 `gbrain jobs dashboard`。查看每个平台上的每个代理。
它们的进度、工具调用、令牌消耗。点击任何代理查看完整执行跟踪。
输入消息以在飞行中重定向运行中的代理。可视化查看管理者的决策。
在代理配置之间运行 A/B 测试。感觉：对您的 AI 劳动力具有完整的态势感知。

**注意：** 仪表板、A/B 测试和可视化管理者是未来阶段。本计划
构建它们将依赖的原语：实时事件、结构化进度、
令牌计费、带确认的收件箱和会话记录。

## 范围决策

| # | 提案 | 工作量 | 决策 | 理由 |
|---|----------|--------|----------|-----------|
| 1 | pg LISTEN/NOTIFY 实时事件 | S | ACCEPTED | 亚秒级事件传递 vs 5秒轮询。每个平台都受益。 |
| 2 | 结构化进度协议 | S | ACCEPTED | 标准进度使统一仪表板成为可能。 |
| 3 | 作业成本跟踪（令牌计费） | M | ACCEPTED | 令牌成本是用户想了解代理工作的 #1 事情。 |
| 4 | 作业重放 | S | ACCEPTED | 表面面积小，对调试失败非常有用。 |
| 5 | 作业组 / 波次 | M | DEFERRED | 父子关系已提供分组。存在重叠问题。 |
| 6 | 收件箱确认（已读回执） | S | ACCEPTED | 没有它，收件箱就是 fire-and-forget — 与我们要解决的问题相同。 |
| 7 | 通用代理协议 | S | ACCEPTED | 设计框架，不是额外代码。平台无关命名/文档。 |
| 8 | 会话记录捕获 | M | ACCEPTED | 每次代理运行的完整审计跟踪。 |

## 接受范围 — 实现细节

### 0a. 暂停/恢复（来自基础计划）

**模式：** 向 `MinionJobStatus` 添加 `'paused'`（已在迁移 v6 约束中）。

**新方法：**
- `MinionQueue.pauseJob(id): MinionJob | null`
  将 `waiting` 或 `active` 状态转换为 `paused`。对于 `active` 作业，清除 `lock_token`
  和 `lock_until`（工作器将检测到锁丢失并停止）。如果作业不在可暂停状态，返回 null。
- `MinionQueue.resumeJob(id): MinionJob | null`
  将 `paused` 状态转换为 `waiting`。重置以进行认领。如果未暂停，返回 null。

**工作器集成：** 工作器的锁更新循环检查 `isActive()`。当作业
暂停时，锁被清除，因此 `renewLock()` 返回 false，工作器优雅地停止
执行（与停顿检测相同的路径）。作业的进度和状态在恢复时保留在数据库中。

**MCP 操作：** `pause_job`、`resume_job`（在实现计划的步骤 3 中添加）。

**PGLite 兼容性：** 完全支持。

### 0b. 资源管理者（来自基础计划）

**新文件：** `src/core/minions/governor.ts`

```typescript
interface GovernorConfig {
  maxConcurrency: number;       // 上限
  minConcurrency: number;       // 下限（默认 1）
  checkIntervalMs: number;      // 默认 10000
  cpuThreshold: number;         // 默认 0.80 (80%)
  memoryThreshold: number;      // 默认 0.85 (85%)
  circuitBreakerMemory: number; // 默认 0.90 (90%)
}

class ResourceGovernor {
  getEffectiveConcurrency(): number;  // 当前允许的并发数
  start(): void;                       // 开始轮询系统指标
  stop(): void;                        // 停止轮询
  onCircuitBreak(cb: (jobId) => void): void; // 终止回调
}
```

**系统指标：** 重用 `src/core/backoff.ts` 中的 `getSystemLoad()`（已
实现 CPU 和内存检查）。通过 `perf_hooks.monitorEventLoopDelay()` 添加事件循环延迟测量。

**工作器集成：** `MinionWorker.start()` 在认领新作业前咨询 `governor.getEffectiveConcurrency()`。
如果当前进行中的计数 >= 有效并发数，跳过认领。

**熔断：** 如果内存 > 90%，管理者使用最低优先级的活动作业 ID 调用 `onCircuitBreak`。工作器通过 `failJob()` 取消该作业，并带有
`UnrecoverableError("circuit breaker: memory pressure")`。

**前提条件：** 必须首先实现并发作业处理（见下方并发说明）。

**PGLite 兼容性：** 完全支持（管理者是应用级，不是数据库级）。

### 1. pg LISTEN/NOTIFY（实时事件）

**模式：** 无新列。向状态转换添加 NOTIFY 触发器。

**SQL 触发器：**
```sql
CREATE OR REPLACE FUNCTION notify_minion_job_change() RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify('minion_jobs', json_build_object(
    'id', NEW.id, 'status', NEW.status, 'name', NEW.name,
    'queue', NEW.queue, 'prev_status', COALESCE(OLD.status, 'new')
  )::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER minion_job_notify AFTER INSERT OR UPDATE OF status ON minion_jobs
  FOR EACH ROW EXECUTE FUNCTION notify_minion_job_change();
```

**新方法：** `MinionQueue.subscribe(callback: (event) => void): () => void`
返回取消订阅函数。需要直接 Postgres 连接（非池化）。

**PGLite 兼容性：** PGLite 不支持 LISTEN/NOTIFY。回退：通过 `getJob()` 以可配置间隔（默认 2秒）轮询。`subscribe()` 方法
检测引擎类型并自动使用轮询回退。

**Supabase 约束：** 需要直接连接（端口 5432），而不是 pgBouncer
池化器（端口 6543）。在技能文件和设置指南中记录。

### 2. 结构化进度协议

**TypeScript 接口（约定，不在数据库级别强制执行）：**
```typescript
interface AgentProgress {
  step: number;           // 当前步骤（从1开始）
  total: number;          // 预期总步骤（0 = 未知）
  message: string;        // 人类可读状态
  tokens_in: number;      // 累积输入令牌
  tokens_out: number;     // 累积输出令牌
  last_tool: string;      // 最后调用的工具名称
  started_at: string;     // 此步骤开始的 ISO 8601 时间
}
```

**存储：** 现有 `progress JSONB` 列。无需更改模式。
处理器使用 `ctx.updateProgress(agentProgress)`。非代理作业可以使用
任何 JSONB 形状（向后兼容）。

**验证：** `updateProgress()` 接受任何 JSONB。`AgentProgress`
接口是由代理处理器强制执行的约定，而不是队列。

### 3. 作业成本跟踪（令牌计费）

**模式更改（迁移 v6）：**
```sql
ALTER TABLE minion_jobs ADD COLUMN tokens_input INTEGER DEFAULT 0;
ALTER TABLE minion_jobs ADD COLUMN tokens_output INTEGER DEFAULT 0;
ALTER TABLE minion_jobs ADD COLUMN tokens_cache_read INTEGER DEFAULT 0;
ALTER TABLE minion_jobs ADD COLUMN cost_usd NUMERIC(10,6) DEFAULT 0;
```

**新方法：** `MinionQueue.updateTokens(id, lockToken, { input, output, cache_read, cost_usd })`
累积（添加到现有值，不替换）。

**父级汇总：** 调用 `completeJob()` 时，如果设置了 `parent_job_id`，
通过以下方式将此作业的令牌计数添加到父级：
```sql
UPDATE minion_jobs SET
  tokens_input = tokens_input + $child_input,
  tokens_output = tokens_output + $child_output,
  tokens_cache_read = tokens_cache_read + $child_cache,
  cost_usd = cost_usd + $child_cost
WHERE id = $parent_id;
```

**PGLite 兼容性：** 完全支持（标准列）。

### 4. 作业重放

**新方法：** `MinionQueue.replayJob(id, dataOverrides?: Record<string, unknown>): MinionJob`

实现：读取已完成/失败/死亡的作业。创建一个**新**作业，包含：
- 相同的 `name`、`queue`、`priority`、`max_attempts`、`backoff_type`、`backoff_delay`
- `data` = 原始数据 + 覆盖的深度合并
- 新的 `attempts_made: 0`，`status: 'waiting'`
- `parent_job_id` = null（重放是新的顶级作业，不是子作业）
- **不**克隆子作业（重放是单个作业，不是 DAG）

**约束：** 仅适用于终端状态（已完成/失败/死亡）。
返回新作业记录。

**幂等性：** 每次重放创建一个不同的新作业。无去重。
如果原始作业有副作用，重放可能会重复它们。在技能文件中将此记录为用户责任。

### 5. 收件箱（侧通道消息）

**模式更改（迁移 v6）：**
```sql
ALTER TABLE minion_jobs ADD COLUMN inbox JSONB DEFAULT '[]';
```

**收件箱消息格式：**
```typescript
interface InboxMessage {
  id: string;          // UUIDv4
  sent_at: string;     // ISO 8601
  read_at: string | null;  // null 直到工作器读取
  sender: string;      // 'parent' | 'user' | 作业 ID
  payload: unknown;    // 任意指令
}
```

**新方法：**
- `MinionQueue.sendMessage(jobId, payload, sender?): InboxMessage`
  通过原子 JSONB 追加将消息追加到收件箱数组
  (`inbox = inbox || $1::jsonb`)，而不是读取-修改-写入。返回带 id + sent_at 的消息。
- `MinionQueue.readInbox(jobId, lockToken): InboxMessage[]`
  返回未读消息（read_at = null）。标记为已读（设置 read_at）。
  令牌保护：只有持有锁的工作器才能读取。

**工作器集成：** 代理处理器在每次迭代时调用 `readInbox()`。
如果存在消息，将其作为系统消息注入代理的上下文中。

**PGLite 兼容性：** 完全支持（标准 JSONB 列）。

### 6. 收件箱确认（已读回执）

内置在上面的收件箱设计中。每个 `InboxMessage` 上的 `read_at` 字段
提供回执。`sendMessage()` 返回消息 ID；发送者稍后可以
检查 `getJob(id)` 并检查 `inbox` 以查看哪些消息已被读取。

除了 #5 中的内容外，不需要额外的模式或方法。

### 7. 通用代理协议（平台无关框架）

**这是设计决策，不是代码。** 这意味着：

1. 技能文件（`skills/minion-orchestrator/SKILL.md`）是为**任何**
   代理平台编写的，不仅仅是 OpenClaw。示例显示 MCP 工具调用，而不是
   OpenClaw 特定命令。

2. 代理处理器（`agent-handler.ts`）接受通用接口：
   ```typescript
   interface AgentJobData {
     prompt: string;
     tools?: string[];        // MCP 工具名称
     model?: string;          // e.g., 'claude-opus-4-6', 'gpt-4o'
     context?: string;        // 额外上下文
     platform?: string;       // 'openclaw' | 'hermes' | 'claude-code' | 'custom'
     max_iterations?: number; // 代理循环预算
   }
   ```

3. OpenClaw 插件是**一个**消费者。Hermes、Claude Code 扩展或自定义脚本可以通过相同的 MCP 操作提交 `agent` 作业。

4. **不在 v1 范围内：** 多租户认证、跨网络连接、协议版本控制、API 密钥隔离。这些是 Phase 2 的关注点，当实际多平台使用出现时。v1 是单用户、单大脑。

### 代理处理器架构（关键设计决策）

代理处理器**不**存在于 GBrain 中。GBrain 提供队列基础设施和干净的处理器契约。实际的代理执行存在于平台插件中。

```
GBrain (本仓库):
  MinionQueue  — queue/claim/complete/inbox/tokens/NOTIFY
  MinionWorker — poll/lock/stall/governor 框架
  Handler contract — AgentJobData 接口 + MinionJobContext

OpenClaw 插件（单独仓库）:
  向 MinionWorker 注册 "agent" 处理器
  处理器调用 OpenClaw 的 PI 代理核心（实际的 LLM 循环）
  每次迭代：readInbox → 注入为系统消息，updateProgress，updateTokens
  完成：将结果 + 会话记录存储在 job.result + job.stacktrace 中

GBrain 仅为单元测试提供测试/回显处理器。
```

**处理器契约（GBrain 端）：**
```typescript
// 处理器接收此上下文（已存在于 worker.ts 中）
interface MinionJobContext {
  id: number;
  name: string;
  data: Record<string, unknown>;  // 名称为"agent"时为 AgentJobData
  attempts_made: number;
  updateProgress(progress: unknown): Promise<void>;
  updateTokens(tokens: TokenUpdate): Promise<void>;  // NEW
  log(message: string | TranscriptEntry): Promise<void>;
  isActive(): Promise<boolean>;
  readInbox(): Promise<InboxMessage[]>;  // NEW
}
```

**为什么这是正确的：** GBrain 是编排，不是执行。OpenClaw 有 PI 代理核心。Hermes 有 AIAgent。Claude Code 有自己的循环。每个平台带来自己的引擎并注册处理器。GBrain 围绕它管理生命周期、进度、引导、成本跟踪和持久性。

### 8. 会话记录捕获

**扩展现有 stacktrace 机制。** `stacktrace` 字段（字符串的 JSONB 数组）已经捕获日志消息。会话记录使用相同字段但带有结构化条目：

```typescript
type TranscriptEntry =
  | { type: 'log'; message: string; ts: string }
  | { type: 'tool_call'; tool: string; args_size: number; result_size: number; ts: string }
  | { type: 'llm_turn'; model: string; tokens_in: number; tokens_out: number; ts: string }
  | { type: 'error'; message: string; stack?: string; ts: string };
```

**存储：** 现有 `stacktrace JSONB` 列。无需更改模式。
代理处理器附加 `TranscriptEntry` 对象而不是纯字符串。向后兼容：非代理作业继续附加字符串。

**大小问题：** 长时间运行的代理可能会生成大量记录。添加
`max_transcript_entries` 选项（默认 1000），超出时旋转最旧条目（FIFO）。用于取证分析的完整记录可以通过 `gbrain files upload-raw` 存储为大脑文件。

## 模式迁移 v6

所有模式更改都是增量的（ALTER TABLE ADD COLUMN）。无需回填。
现有作业继续使用默认值工作。

```sql
-- Migration v6: Agent orchestration primitives
ALTER TABLE minion_jobs ADD COLUMN IF NOT EXISTS tokens_input INTEGER DEFAULT 0;
ALTER TABLE minion_jobs ADD COLUMN IF NOT EXISTS tokens_output INTEGER DEFAULT 0;
ALTER TABLE minion_jobs ADD COLUMN IF NOT EXISTS tokens_cache_read INTEGER DEFAULT 0;

-- 单独的收件箱表（不是作业行上的 JSONB）
CREATE TABLE IF NOT EXISTS minion_inbox (
  id SERIAL PRIMARY KEY,
  job_id INTEGER NOT NULL REFERENCES minion_jobs(id) ON DELETE CASCADE,
  sender TEXT NOT NULL,
  payload JSONB NOT NULL,
  sent_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  read_at TIMESTAMPTZ
);
CREATE INDEX IF NOT EXISTS idx_minion_inbox_unread
  ON minion_inbox (job_id) WHERE read_at IS NULL;

-- 状态约束更新：添加 'paused'
ALTER TABLE minion_jobs DROP CONSTRAINT IF EXISTS minion_jobs_status_check;
ALTER TABLE minion_jobs ADD CONSTRAINT minion_jobs_status_check
  CHECK (status IN ('waiting','active','completed','failed','delayed','dead','cancelled','waiting-children','paused'));

-- 实时事件的 NOTIFY 触发器（仅 Postgres，非 PGLite）
CREATE OR REPLACE FUNCTION notify_minion_job_change() RETURNS trigger AS $$
BEGIN
  PERFORM pg_notify('minion_jobs', json_build_object(
    'id', NEW.id, 'status', NEW.status, 'name', NEW.name,
    'queue', NEW.queue, 'prev_status', COALESCE(OLD.status, 'new')
  )::text);
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER minion_job_notify AFTER INSERT OR UPDATE OF status ON minion_jobs
  FOR EACH ROW EXECUTE FUNCTION notify_minion_job_change();
```

## PGLite 兼容性矩阵

| 功能 | Postgres | PGLite | 回退 |
|---|---|---|---|
| 暂停/恢复 | 完全 | 完全 | — |
| 收件箱 + 确认 | 完全 | 完全 | — |
| 令牌计费 | 完全 | 完全 | — |
| 作业重放 | 完全 | 完全 | — |
| LISTEN/NOTIFY | 完全 | NO | 轮询（2秒间隔） |
| NOTIFY 触发器 | 完全 | NO | 在 PGLite 模式中跳过 |
| 结构化进度 | 完全 | 完全 | — |
| 会话记录 | 完全 | 完全 | — |
| 资源管理者 | 完全 | 完全 | — |
| 工作器守护进程 | 完全 | NO（现有限制） | — |

## 并发说明

当前的 `MinionWorker.start()` 顺序处理作业（一次一个），尽管 `MinionWorkerOpts` 中声明了 `concurrency`。实现实际的并发作业处理（Promise 池）是资源管理者有意义的前提条件。管理者调整有效并发数，这需要实际的并发处理存在。

**行动：** 在管理者步骤之前或作为其一部分，在 `worker.ts` 中实现并发作业处理。使用信号量模式：维护最多 N 个进行中的 Promise，当槽位空闲时认领新作业。

## 外部意见决策（来自对抗性审查）

1. **暂停/恢复的 AbortController** — 处理器契约获取 `signal: AbortSignal`。
   暂停清除锁并发送中止信号。处理器必须在每次迭代时检查 `signal.aborted`。
   没有这个，暂停活动作业会创建重复执行。

2. **删除 cost_usd 列** — 令牌计数（输入/输出/缓存读取）是稳定事实。
   USD 定价是不稳定的。在显示/读取时从定价表计算成本，而不是写入时。
   从迁移 v6 中删除 `cost_usd NUMERIC(10,6)`。

3. **单独的 minion_inbox 表** — 而不是作业行上的 JSONB 数组，使用专用表存储收件箱消息。
   避免每次发送时重写整个收件箱导致的行膨胀。
   使用标准 INSERT 实现适当的并发安全（无 JSONB 追加问题）。
   ```sql
   CREATE TABLE minion_inbox (
     id SERIAL PRIMARY KEY,
     job_id INTEGER NOT NULL REFERENCES minion_jobs(id) ON DELETE CASCADE,
     sender TEXT NOT NULL,
     payload JSONB NOT NULL,
     sent_at TIMESTAMPTZ NOT NULL DEFAULT now(),
     read_at TIMESTAMPTZ
   );
   CREATE INDEX idx_minion_inbox_unread ON minion_inbox (job_id) WHERE read_at IS NULL;
   ```

4. **一次发布，不是两次** — 在一次迁移（v6）中发布所有功能。用户更喜欢
   这个功能集的内聚发布而不是增量交付。

5. **选择性列投影** — 修复 getJobs()、claim()、handleStalled() 中的 SELECT * 查询，排除 stacktrace 列。仅在 getJob() 详细视图中包含 stacktrace。防止记录膨胀影响查询性能。

## 未来阶段（已接受的轨迹）

- **Phase 2: Dashboard CLI** — `gbrain jobs dashboard` 实时 TUI 显示所有代理。
  启用条件：LISTEN/NOTIFY、结构化进度、令牌计费。
- **Phase 3: 多租户认证** — 运行时 MCP 访问控制、每个平台的 API 密钥。
  启用条件：平台无关框架、收件箱上的发送者验证。
- **Phase 4: 代理组合模式** — Map-reduce、管道、审批门作为一等原语。
  启用条件：父子 DAG、收件箱侧通道。

## 推迟到 TODOS.md
- 作业组 / 波次（父子关系涵盖此；如果出现真正的分组需求，重新考虑）
- cost_usd 列（当定价 API 存在时，在读取时从定价表计算）

## 确认的关键前提
1. GBrain 正有意从知识大脑演变为代理基础设施（用户确认）
2. OpenClaw 和 GBrain 的 Postgres 之间的耦合是可接受的（OpenClaw 已经依赖 GBrain）
3. 完整基础设施方法（所有 8+ 步骤）被选中，而不是最小可行或侧车跟踪
4. 先前的学习 [agent-dx-instruction-layer] 验证了教学层（技能 + 评估）是强制性的