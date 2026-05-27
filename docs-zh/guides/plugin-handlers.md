# 插件处理器 — 注册主机特定的 Minion 处理器

GBrain 的 Minion 工作进程附带七个内置处理器：`sync`、`embed`、`lint`、`import`、`extract`、`backlinks`、`autopilot-cycle`。这些涵盖了 gbrain CLI 本身执行的所有后台操作。

主机平台（OpenClaw 部署、未来的主机）通过导入 `gbrain/minions` 的插件引导程序注册自己的处理器。没有 `handlers.json` 风格的数据文件 — 处理器是代码，由工作进程加载，具有与主机仓库中任何其他代码相同的信任模型。

## 为什么用代码而不是数据

早期设计草案使用 `~/.claude/gbrain-handlers.json`，其中每个条目都是工作进程在任务声明时执行的 shell 命令。Codex 将其标记为持久的 RCE 表面：一个代理可写的数据文件，能生成任意 shell。我们放弃了数据文件方法；处理器是主机显式导入并通过代码审查发布的代码。

## 插件契约

主机工作进程引导程序如下所示（TypeScript）：

```ts
import { MinionQueue, MinionWorker } from 'gbrain/minions';
import type { BrainEngine } from 'gbrain/engine';

async function main() {
  const engine: BrainEngine = /* your engine setup */;
  await engine.connect({});

  const worker = new MinionWorker(engine, { queue: 'default' });

  // 注册主机 cron 清单引用的每个主机特定处理器。
  // 每个处理器返回一个普通对象（序列化为任务结果）。
  // 失败时抛出异常 — 工作进程会根据 max_attempts 捕获并重试。

  worker.register('ea-inbox-sweep', async (ctx) => {
    const slot = ctx.data.slot ?? new Date().toISOString();
    // 主机特定的代理回合：调用你的 LLM，扫描收件箱，写入
    // 大脑页面，返回摘要。ctx.signal.aborted 表示
    // 工作进程希望你配合关闭 — 请遵守。
    return { swept: true, slot };
  });

  worker.register('morning-briefing', async (ctx) => {
    /* host logic */
    return { briefed: true };
  });

  // 在所有处理器注册完成后调用 start()。工作进程的
  // 停滞检测器会忽略名称不在注册集合中的任务。
  await worker.start();
}

main().catch(err => { console.error(err); process.exit(1); });
```

将此作为主机仓库中的单独二进制文件（例如 `your-openclaw-worker`）发布，或作为库存 `gbrain jobs work` 命令在启动时自动加载的副作用模块（可通过主机提供的入口点配置）。

## 处理器契约

每个处理器接收一个 `MinionJobContext`：

```ts
interface MinionJobContext {
  data: Record<string, unknown>;   // 任务参数（cron 提交传递的任何内容）
  job: MinionJob;                   // 完整任务行（id、队列、尝试次数等）
  signal: AbortSignal;              // 工作进程关闭时设置为中止状态
  inbox: MinionInbox;               // 读取任务运行期间发送到此任务的消息
}
```

成功时返回可序列化对象。失败时抛出异常（工作进程将记录并根据 `max_attempts` 重试）。

**中止协作**。当 `ctx.signal.aborted` 变为 true 时，优雅地完成。工作进程将等待 30 秒让你返回，然后发送 SIGKILL。长时间运行的 LLM 调用应将信号传递给它们使用的任何网络库。

**幂等性**。队列在数据库层强制执行唯一的 `idempotency_key`，因此你不必担心前一次调用仍在运行时 cron 触发导致的重复提交。

## GBrain 的迁移流程

v0.11.0 迁移编排器（由 `gbrain apply-migrations` 运行）检测处理器名称不在 GBrain 内置集合中的 cron 条目，并向 `~/.gbrain/migrations/pending-host-work.jsonl` 发出结构化的 TODO。每个 TODO 的格式如下：

```json
{
  "type": "cron-handler-needs-host-registration",
  "handler": "ea-inbox-sweep",
  "cron_schedule": "0 */30 * * *",
  "manifest_path": "/path/to/cron/jobs.json",
  "current_cmd": "agentTurn ea-inbox-sweep",
  "recommendation": "Add a handler registration for `ea-inbox-sweep` in your host worker bootstrap per docs/guides/plugin-handlers.md. Once registered, re-run `gbrain apply-migrations` to auto-rewrite this entry.",
  "status": "pending"
}
```

主机代理使用 `skills/migrations/v0.11.0.md` 处理这些条目：

1. 读取 `~/.gbrain/migrations/pending-host-work.jsonl`。
2. 对于每个 `cron-handler-needs-host-registration` 行，按照上述模式在主机的工作进程引导程序中发布处理器注册。
3. 部署更新后的工作进程。
4. 重新运行 `gbrain apply-migrations --yes`。编排器现在识别新注册的处理器（工作进程在启动时将注册的名称写入发现文件）并重写 cron 条目以使用 `gbrain jobs submit`。JSONL 行标记为 `status: "complete"`。

## 信任边界

处理器代码在工作进程内运行，具有与主机二进制文件其余部分相同的权限。没有权限提升。但也没有运行时沙箱 — 处理器可以读写工作进程用户可以访问的任何地方。以审查任何其他接触生产数据的代码的方式审查处理器 PR。

## 相关文档

- `skills/conventions/cron-via-minions.md` — cron 清单的重写约定。
- `skills/migrations/v0.11.0.md` — 迁移编排器如何驱动主机代理完成此项工作。
- `skills/minion-orchestrator/SKILL.md` — 处理器上线后提交、监控、控制和重放任务的模式。