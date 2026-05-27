# Progress events

`gbrain` 在批量命令使用 `--progress-json` 运行时写入 `stderr` 的 JSONL 进度流的规范参考。从 v0.15.2 开始稳定。仅允许增量更改；不进行重命名或删除，除非进行重大版本升级。

大多数人不会阅读此页面。解析进度的代理会阅读。

## 何时获得这些事件？

设置 `--progress-json` 时，以下任何命令都会流式传输事件：

- `gbrain doctor`（数据库检查、JSONB 完整性、markdown 正文完整性、完整性样本）
- `gbrain orphans`
- `gbrain embed`
- `gbrain files sync`
- `gbrain export`
- `gbrain extract [links|timeline|all]`（文件系统或数据库源）
- `gbrain import`
- `gbrain sync`
- `gbrain migrate --to …`
- `gbrain repair-jsonb`
- `gbrain check-backlinks`
- `gbrain lint`
- `gbrain integrity auto`
- `gbrain eval`
- `gbrain apply-migrations`（编排器 + 每个子命令）

非批量命令（`stats`、`graph-query`、`get`、`put` 等）不发出事件——它们在不到一秒钟内返回。

## 通道

- 进度事件：**`stderr`**，每行一个 JSON 对象，`\n` 终止。
- 数据结果（每个命令的 `--json` 有效负载）：**`stdout`**。
- 最终人类摘要：**`stdout`**。

代理可以安全地捕获 stdout 进行结果解析，并单独读取 stderr 获取进度。

## 标志

| 标志 | 行为 |
|---|---|
| *(none)* | 自动。TTY：`\r` 重写单行。非 TTY：stderr 上的简单每行事件。 |
| `--progress-json` | 在 stderr 上强制使用 JSON-lines 模式（本文档）。 |
| `--quiet` | 完全抑制进度。警告和最终输出仍会打印。 |
| `--progress-interval=<ms>` | 覆盖 tick 发射之间的最小间隔（默认 1000）。 |

全局标志：在命令分派之前由 `src/core/cli-options.ts` 解析，因此 `gbrain --progress-json doctor` 与 `gbrain doctor --progress-json` 工作方式相同（后者也有效——每个命令的解析器通过共享的 `CliOptions` 单例看到标志）。

## 事件类型

每个事件都是一个单行 JSON 对象，包含以下公共字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `event` | string | 以下之一：`start`、`tick`、`heartbeat`、`finish`、`abort`。 |
| `phase` | string | 机器稳定的 snake_case，点分隔。见下文"阶段名称"。 |
| `ts` | ISO 8601 UTC string | 事件发射时间。 |
| `elapsed_ms` | number | 自阶段开始以来的毫秒数。存在于 `tick`/`heartbeat`/`finish`/`abort`。 |

### `start`

阶段开始时发出。

```json
{"event":"start","phase":"doctor.db_checks","ts":"2026-04-20T12:34:56.789Z"}
{"event":"start","phase":"import.files","total":52000,"ts":"2026-04-20T12:34:56.789Z"}
```

可选字段：

- `total` — 如果在开始时已知，则为总项目数。

### `tick`

在迭代期间定期发出。受时间和项目限制：报告器不会比 `minIntervalMs`（默认 1000）和 `minItems`（默认 `max(10, ceil(total/100))`）更频繁地发出。

```json
{"event":"tick","phase":"orphans.scan","done":15000,"total":52000,"pct":28.8,"elapsed_ms":4200,"eta_ms":10300,"ts":"..."}
```

字段：

- `done` — 此阶段完成的项目数。
- `total` — 总项目数（如果已知）。当扫描事先没有总数时省略（例如流式迭代器）。
- `pct` — `done/total * 100`，一位小数。当 `total` 未知时省略。
- `eta_ms` — 根据观察到的速率，预计到 `done === total` 的毫秒数。当 `total` 未知时省略。
- `note` — 可选字符串，包含当前项目（例如 slug 或文件名）。

### `heartbeat`

为长时间运行的单个操作发出，这些操作不迭代（例如针对 50K 行表的 `SELECT`）。没有 `done`，没有 `total`——只是工作仍在进行的信号。

```json
{"event":"heartbeat","phase":"doctor.markdown_body_completeness","note":"scanning pages for truncation…","elapsed_ms":1000,"ts":"..."}
```

### `finish`

阶段正常完成时发出。

```json
{"event":"finish","phase":"import.files","done":52000,"total":52000,"elapsed_ms":187000,"ts":"..."}
```

### `abort`

由跟踪每个活动阶段的单个进程级 SIGINT/SIGTERM 处理程序发出。`abort` 后，该阶段不再发出进一步的事件。

```json
{"event":"abort","phase":"doctor.markdown_body_completeness","reason":"SIGINT","elapsed_ms":5300,"ts":"..."}
```

## 阶段名称

阶段使用 `snake_case.dot.path` 命名。新的报告器从根开始；`child()` 组合附加到父级的当前阶段，因此调用 import 的 sync 发出 `sync.import.<file>`，而不是 `import.<file>`。

v0.15.2 中发布的稳定阶段名称：

- `doctor.db_checks`（所有数据库端医生检查的总括）
- `orphans.scan`
- `embed.pages`
- `extract.links_fs`、`extract.timeline_fs`、`extract.links_db`、`extract.timeline_db`
- `import.files`
- `sync.deletes`、`sync.renames`、`sync.imports`
- `migrate.copy_pages`、`migrate.copy_links`
- `repair_jsonb.run`、`repair_jsonb.<table>.<column>`
- `backlinks.scan`
- `lint.pages`
- `integrity.auto`
- `eval.single`、`eval.ab`
- `export.pages`
- `files.sync`

通过 `child()` 暴露的子阶段：

- `sync.import.files` — 嵌套在 sync 内
- `apply_migrations.v0_12_2.jsonb_repair` — 嵌套在编排器内

## 子进程继承

当父 CLI 生成 `gbrain …` 子进程时（主要在 `src/commands/migrations/*` 中），全局标志（`--quiet`、`--progress-json`、`--progress-interval`）通过 `src/core/cli-options.ts` 中的 `childGlobalFlags()` 辅助函数传播到子进程的 argv。子进程的 stderr 通过 `stdio: 'inherit'` 直接传递，因此事件流是父进程 stderr 上的一个合并的 JSONL feed。

一个例外：`migrations/v0_12_2.ts` 中的编排器阶段捕获子进程的 stdout（`repair-jsonb --dry-run --json` 用于验证），不传递 `--progress-json` 以避免 stdout 污染破坏编排器的 `JSON.parse`。其 stdio 是显式的：`['ignore', 'pipe', 'inherit']`，因此 stderr 仍然流过。

## Minion jobs

`gbrain jobs work`（Minion worker daemon）将进度保存在数据库中，而不是 stderr 上。每个运行批量核心（embed、sync、extract、import、backlinks）的 Minion 处理程序在每次迭代时调用 `job.updateProgress({done, total, …})`。代理通过 `get_job_progress` MCP 操作或 `gbrain jobs get <id>` 读取每个作业的进度。

`jobs work` 守护进程本身只为了活跃度在 stderr 上发出粗略的每行一个作业的输出。每页的详细信息存储在数据库中。

## 兼容性

- **添加**：仅添加。新的事件类型、新字段、新阶段名称——所有都是安全的。代理必须忽略未知字段和未知事件类型。
- **删除/重命名**：除非进行重大版本升级，否则永不删除/重命名。
- **Schema 更改**：在 `CHANGELOG.md` 和 `skills/migrations/v<next>.md` 中宣布。

如果您的代理依赖于此 schema 并且有什么让您感到惊讶，请打开一个 issue，说明您收到的事件和您的预期。