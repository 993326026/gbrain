# Minions shell jobs — 将确定性 cron 从网关移走

## 30秒快速入门

```bash
# 运行你的第一个 shell 任务：
GBRAIN_ALLOW_SHELL_JOBS=1 gbrain jobs submit shell \
  --params '{"cmd":"echo hello","cwd":"/tmp"}' --follow
# → exit_code: 0, stdout_tail: "hello\n", duration_ms: 43
```

就这样。你的 cron 脚本现在有了一个家，带有重试、退避、DLQ 和 `gbrain jobs list` 可见性，而不需要启动完整的 LLM 会话。

**PGLite 用户：** `gbrain jobs work` 不在 PGLite 上运行（独占文件锁）。每个 crontab 调用必须使用 `--follow` 进行内联执行。Postgres 用户可以运行持久 worker；请参阅下面的配方。

---

## 为什么存在

如果你的代理从 cron 运行确定性脚本（令牌刷新、API 获取、抓取 + 写入），每个脚本都要在网关上支付完整 LLM 会话的成本。在 A 轮部署上，14 个同时触发会使 CPU 固定在 100% 并阻塞实时消息。这些脚本都不需要推理。它们需要一个 shell。

Shell jobs 将它们移到 Minions worker：每个 cron 一个确定性脚本执行，零 LLM token，统一的可见性和重试。

---

## 安全模型（阅读此部分）

Shell 执行的影响范围很大。我们提供两个独立的门，都必须通过：

1. **MCP 边界。** 当 `ctx.remote === true`（MCP 调用者）时，`submit_job` 使用 `name: 'shell'` 会被拒绝。与环境标志无关。远程代理永远不能提交 shell 任务。`MinionQueue.add('shell', ...)` 也有自己的守卫，所以进程内处理程序不能编程式绕过这个限制。
2. **环境标志。** 只有当 `GBRAIN_ALLOW_SHELL_JOBS=1` 设置在 worker 进程上时，worker 才会注册 shell 处理程序。默认：关闭。你的代理按主机选择加入。

**环境允许列表的作用和不作用。** Shell jobs 在最小环境中运行：`PATH, HOME, USER, LANG, TZ, NODE_ENV`。你的 secrets 如 `OPENAI_API_KEY` 和 `DATABASE_URL` **不会**传递给子进程。你通过 `env: { ... }` 按任务选择加入其他键。这可以防止用户编写的脚本中意外的 `$OPENAI_API_KEY` 插值。它**不会**沙箱化文件系统读取：shell 脚本可以 `cat ~/.env` 或 worker 进程可以读取的任何文件。操作员选择安全的 `cwd`。这是信任边界。

**审计追踪，不是取证保险。** 每次提交都会向 `~/.gbrain/audit/shell-jobs-YYYY-Www.jsonl` 写入一行 JSONL（ISO 周轮换；用 `GBRAIN_AUDIT_DIR` 覆盖）。失败记录到 stderr，不阻塞提交，所以磁盘满的攻击者可以静默禁用追踪。适用于"上周二这个 cron 提交了什么"，不适用于安全关键取证。

**命令文本按原样记录。** 如果你在 `cmd` 中嵌入 secret（`curl -H 'Authorization: Bearer ...'`），它会出现在审计文件中。改为将 secrets 放在 `env:` 中。

---

## 迁移 cron

### Postgres worker（推荐）

在一个终端中，启动持久 worker：

```bash
GBRAIN_ALLOW_SHELL_JOBS=1 gbrain jobs work
```

重写 crontab 以提交 shell jobs（无 `--follow`）：

```cron
# 之前（LLM 网关）：
#   OpenClaw cron: x-garrytan-unified
# 之后（Minions worker）：
3 13,16,19,22,1,4,7,10 * * * \
  gbrain jobs submit shell \
    --params '{"cmd":"node scripts/x-garrytan-daily.mjs","cwd":"/data/.openclaw/workspace"}' \
    --max-attempts 3 --timeout-ms 300000
```

Worker 在下次轮询时获取任务，运行它，在结果中记录 `exit_code` + `stdout_tail` + `stderr_tail`。失败按 `--max-attempts` 重试，带指数退避。

### PGLite（内联执行）

PGLite 不支持持久 worker 守护进程。每个 crontab 调用使用 `--follow` 内联运行：

```cron
# 每个 cron tick 生成一个短期 worker，内联运行任务。
3 13,16,19,22,1,4,7,10 * * * \
  GBRAIN_ALLOW_SHELL_JOBS=1 gbrain jobs submit shell \
    --params '{"cmd":"node scripts/x-garrytan-daily.mjs","cwd":"/data/.openclaw/workspace"}' \
    --follow --timeout-ms 300000
```

注意：`--follow` 阻塞 crontab 槽直到任务完成。如果 14 个 shell cron 在同一分钟到达，每个需要 30 秒，它们会通过 crontab 的生成限制序列化。Postgres + 持久 worker 扩展性更好。

### 使用 `argv` 提交（无 shell 插值）

对于从 JSON 组装命令的编程调用者，使用 `argv` 而不是 `cmd`。无 shell，无注入面：

```bash
gbrain jobs submit shell \
  --params '{"argv":["node","scripts/fetch.mjs","--date","2026-04-19"],"cwd":"/data"}' \
  --follow
```

---

## 调试失败的任务

```bash
# 列出死信 shell jobs
gbrain jobs list --status dead

# 检查一个任务
gbrain jobs get 42
# → error_text, stacktrace, result.stdout_tail, result.stderr_tail

# 提交审计日志（操作员追踪，非取证）
cat ~/.gbrain/audit/shell-jobs-*.jsonl | jq '.'

# 首次失败模式：在 worker 上没有设置环境标志就提交
gbrain jobs list --status waiting --name shell
# 如果行堆积在这里，没有运行带 GBRAIN_ALLOW_SHELL_JOBS=1 的 worker。
```

---

## 限制

- **文件系统读取未沙箱化。** 见上面的"安全模型"。不要将 `cwd` 指向充满 secrets 的目录。
- **审计日志是建议性的。** 磁盘满或 EACCES 会静默禁用它。
- **取消延迟受锁续约限制**（默认约 7-15 秒）。被取消的子进程继续运行直到下一次锁续约失败。
- **`--follow` 获取顺序** 按 priority/created_at。如果 `--follow` 时同一队列中有另一个任务在等待，那个任务先运行。
- **`cwd` 符号链接 TOCTOU。** 绝对路径检查不保护执行时符号链接指向其他地方。操作员范围的关注点。

---

## 错误 {#errors}

| 错误 | 含义 | 修复 |
|---|---|---|
| `shell: specify exactly one of cmd or argv` | `cmd` 和 `argv` 互斥。两者都缺少也是无效的。 | 选择一个。`cmd` 用于 shell 插值字符串；`argv` 用于结构化参数。 |
| `shell: cwd is required and must be an absolute path` | `cwd` 必须是以 `/` 开头的字符串。 | 在 `--params` 中将 `cwd` 设置为绝对路径。 |
| `shell: argv must be an array of strings` | `argv` 有非字符串条目或不是数组。 | 传递 `argv: ["bin","arg1","arg2"]`。 |
| `shell: env values must all be strings` | `env` 有数字/布尔/对象值。 | 字符串化：`"env":{"COUNT":"3"}` 而不是 `"env":{"COUNT":3}`。 |
| `permission_denied: shell jobs cannot be submitted over MCP` | MCP 客户端尝试提交 shell job。设计上仅 CLI。 | 从 CLI 提交或通过可信操作处理程序（`ctx.remote === false`）。 |
| `protected job name 'shell' requires CLI or operation-local submitter` | 调用者在没有 `trusted` 选择加入的情况下调用 `MinionQueue.add('shell', ...)`。 | 作为第 4 个参数传递 `{ allowProtectedSubmit: true }`。CLI 和 `submit_job` 自动执行此操作。 |
| `aborted: timeout` / `aborted: cancel` / `aborted: shutdown` / `aborted: lock-lost` | worker 的中止信号在执行过程中触发。子进程收到 SIGTERM，5 秒宽限期，然后 SIGKILL。 | 预期：超时 / 用户取消 / 部署重启 / 停顿。检查 `gbrain jobs get` 查看是哪一个。 |
| `exit N: <stderr_tail_500>` | 脚本非零退出。 | 在 `gbrain jobs get` 中读取 `stderr_tail`。 |