# Minions Worker 部署指南

在崩溃、重启和 Postgres 连接中断时保持 `gbrain jobs work` 运行。为代理逐行执行而编写。

## 问题

持久 worker 可能会无声死亡，原因包括：

- 数据库连接断开（Supabase/Postgres 维护或网络中断）。
- 锁续约失败 → 停顿检测器最终将任务死信。
- Bun 进程崩溃，没有自动重启。
- 内部事件循环死亡（PID 存活，但 worker 循环停止）。

当 worker 死亡时，提交的任务永远停留在 `waiting` 状态。标准解决方案是 `gbrain jobs supervisor` —— 一个一流的 CLI，生成 `gbrain jobs work` 作为子进程，并在崩溃时自动重启。

## Worker 监控

### 标准模式

`gbrain jobs supervisor` 是围绕 `gbrain jobs work` 的自动重启包装器。它写入 PID 文件，在崩溃时使用指数退避重启 worker（1秒 → 60秒上限），向审计文件发出生命周期事件，并在 SIGTERM 上优雅排空（35秒 worker 排空窗口，然后 SIGKILL）。退出代码已记录，以便代理可以根据它们进行分支。

**典型命令：**

```bash
# 前台启动（阻塞；Ctrl-C 停止）。
gbrain jobs supervisor --concurrency 4

# 分离启动 —— 在 stdout 上返回 {"event":"started","supervisor_pid":…}。
gbrain jobs supervisor start --detach --json

# 检查活跃度，无需读取日志文件。
gbrain jobs supervisor status --json

# 优雅停止（SIGTERM + 排空等待 + SIGKILL 回退）。
gbrain jobs supervisor stop
```

**退出代码：**

| 代码 | 含义 |
|---|---|
| 0 | 干净关闭（收到 SIGTERM/SIGINT，worker 已排空） |
| 1 | 超过最大崩溃次数（worker 持续死亡） |
| 2 | 另一个 supervisor 持有 PID 锁 |
| 3 | PID 文件不可写（权限/路径错误） |

代理看到 exit=2 可以安全地将其视为"已有一个正在运行"；exit=1 应该通知人工。

### 何时使用哪种 supervisor

Supervisor 解决进程内崩溃恢复。平台级监控（systemd、Fly、Render）处理主机级故障。通常两者都需要。

| 环境 | 建议 |
|---|---|
| **容器（Fly / Railway / Render / Heroku）** | `gbrain jobs supervisor` 作为 PID 1 运行。平台在 OOM/主机丢失时重启容器；supervisor 在崩溃时重启 worker。参见 [Fly.io](#flyio) / [Render / Railway / Heroku](#render--railway--heroku)。 |
| **带 systemd 的 Linux VM** | 建议两层：systemd 监控 `gbrain jobs supervisor`，后者又监控 `gbrain jobs work`。获得重启时自动重启（systemd）加上快速崩溃恢复（supervisor）。参见 [systemd](#systemd)。 |
| **开发笔记本电脑 / macOS** | 在终端中运行 `gbrain jobs supervisor`。Ctrl-C 停止。无需系统级设置。 |

### 本指南中使用的变量

在复制粘贴任何代码段之前先替换这些变量。

| 变量 | 含义 | 典型值 |
|---|---|---|
| `$GBRAIN_BIN` | `gbrain` 二进制文件的绝对路径 | `$(command -v gbrain)` — 通常是 `/usr/local/bin/gbrain` 或 `~/.bun/bin/gbrain` |
| `$GBRAIN_WORKER_USER` | 拥有 worker 进程的 OS 用户 | 运行 `gbrain init` 的同一用户；永远不要用 `root` |
| `$GBRAIN_WORKSPACE` | 此部署提交的 shell 任务的 `cwd` | 绝对路径，例如 `/srv/my-brain` |
| `$GBRAIN_ENV_FILE` | systemd / shell 读取的 secrets 文件 | `/etc/gbrain.env`（模式 600） |

### 前提条件

在任何部署步骤之前运行这些。

```bash
# 1. gbrain 在 PATH 上并且解析到绝对位置。
command -v gbrain || { echo "gbrain not on PATH. Install, then retry."; exit 1; }

# 2. DATABASE_URL 指向可访问的 Postgres。
#    (Supervisor 仅支持 Postgres。PGLite 的独占文件锁会阻止
#    单独的 worker 进程。如果 `config.engine === 'pglite'`，CLI 会以
#    清晰的错误拒绝。)
gbrain doctor --fast --json | jq '.checks[] | select(.name=="db_connectivity")'

# 3. Schema 是最新的。如果 version=0 或 status=="fail"：
#    gbrain apply-migrations --yes
gbrain doctor --fast --json | jq '.checks[] | select(.name=="schema_version")'

# 4. 如果计划提交 `shell` 任务，向 supervisor 传递 --allow-shell-jobs
#    (或在启动前导出 GBRAIN_ALLOW_SHELL_JOBS=1)。
#    没有此标志，shell 处理器在 worker 启动时被禁用。
```

## 代理使用（OpenClaw / Hermes / Cursor / Codex）

代理无需 shell 考古就能驱动的三命令模式：

```bash
# 启动（在 stdout 上返回 PIDs + pid_file 作为 JSON，然后分离）
gbrain jobs supervisor start --detach --json
# → {"event":"started","supervisor_pid":1234,"worker_pid":1235,"pid_file":"/Users/you/.gbrain/supervisor.pid"}

# 检查健康（机器可解析的 JSON，无需日志抓取）
gbrain jobs supervisor status --json
# → {"running":true,"supervisor_pid":1234,"last_start":"2026-04-23T15:30:22Z","crashes_24h":0, ...}

# 干净停止（SIGTERM + 35秒排空 + SIGKILL 回退）
gbrain jobs supervisor stop
```

每个生命周期事件（生成、崩溃、退避、健康警告、最大崩溃次数、关闭）也会写入 `${GBRAIN_AUDIT_DIR:-~/.gbrain/audit}/supervisor-YYYY-Www.jsonl` 以供历史检查。`gbrain doctor` 读取该文件并在其健康报告中显示 `supervisor` 检查。

## 部署：systemd

适用于具有 shell 访问权限的长期运行的 Linux VM。

```bash
# 如果 worker 用户不存在，则创建它。
sudo useradd --system --home "$GBRAIN_WORKSPACE" --shell /usr/sbin/nologin gbrain \
  2>/dev/null || true
sudo mkdir -p "$GBRAIN_WORKSPACE" && sudo chown gbrain:gbrain "$GBRAIN_WORKSPACE"

# 安装 env 文件（secrets 不进入单元文件）。
sudo install -m 600 -o gbrain -g gbrain \
  docs/guides/minions-deployment-snippets/gbrain.env.example /etc/gbrain.env
sudoedit /etc/gbrain.env
# 填写 DATABASE_URL，可选 GBRAIN_ALLOW_SHELL_JOBS=1。

# 安装单元文件，将 /srv/gbrain 替换为你的工作空间路径。
sudo install -m 644 docs/guides/minions-deployment-snippets/systemd.service \
  /etc/systemd/system/gbrain-worker.service
sudo sed -i "s|/srv/gbrain|$GBRAIN_WORKSPACE|g" \
  /etc/systemd/system/gbrain-worker.service

sudo systemctl daemon-reload
sudo systemctl enable --now gbrain-worker
sudo systemctl status gbrain-worker
journalctl -u gbrain-worker -n 50
```

附带的单元文件调用 `gbrain jobs supervisor`（而不是直接调用 `gbrain jobs work`），因此你获得两层监控：systemd 在主机重启时重启 supervisor，supervisor 在进程内崩溃时重启 worker。

`Restart=always` + `RestartSec=10s` 处理 supervisor 级别的恢复。单元以非特权 `gbrain` 用户运行，具有 `PrivateTmp`、`ProtectSystem=strict` 和 `ReadWritePaths=$GBRAIN_WORKSPACE,$HOME/.gbrain`（用于 PID 文件和审计日志）。`LimitNOFILE=65535` 覆盖 Bun + Postgres 池 + 并发 LLM 子代理调用，不会达到默认的 1024 限制。

## 部署：Fly.io

```bash
# 将 fly.toml.partial 中的 [processes] 块合并到你的 fly.toml 中。
cat docs/guides/minions-deployment-snippets/fly.toml.partial >> fly.toml
# 根据需要查看和编辑。

# 设置 secrets（Fly 处理崩溃重启）。
fly secrets set DATABASE_URL='postgres://…' GBRAIN_ALLOW_SHELL_JOBS=1
```

`[processes]` 块运行 `gbrain jobs supervisor` 作为 PID 1。Fly 在主机故障时重启容器；supervisor 在进程内崩溃时重启 worker。

## 部署：Render / Railway / Heroku

在仓库根目录放置 [`Procfile`](./minions-deployment-snippets/Procfile)。附带的 Procfile 调用 `gbrain jobs supervisor`。通过平台的环境 UI 或 CLI 设置 `DATABASE_URL` + 可选的 `GBRAIN_ALLOW_SHELL_JOBS=1`。

## 部署：内联 `--follow`（无持久 worker）

适用于固定时间表上的短期确定性脚本，不需要运行之间的持久 worker。每次 cron 运行都会带来自己的临时 worker。`--follow` 在队列上启动一个并阻塞，直到刚刚提交的任务达到终端状态（`completed` / `failed` / `dead` / `cancelled`）。每个任务有 2-3 秒的启动开销；对于计划工作，与任务持续时间相比可以忽略不计。

```bash
GBRAIN_ALLOW_SHELL_JOBS=1 gbrain jobs submit shell \
  --queue nightly-enrich \
  --params "{\"cmd\":\"$GBRAIN_BIN embed --stale\",\"cwd\":\"$GBRAIN_WORKSPACE\"}" \
  --follow \
  --timeout-ms 600000
```

用你要调度的 gbrain 子命令替换 `gbrain embed --stale`（`sync`、`extract`、`orphans`、`doctor`、`check-backlinks`、`lint`、`autopilot`）。为了在共享队列上实现严格的单任务语义，使用像上面的 `nightly-enrich` 这样的专用队列名称。

## 从旧部署升级

### 从 `minion-watchdog.sh`（v0.20 之前）

本指南的早期版本附带了一个 68 行的 bash watchdog（`minion-watchdog.sh`）。它已被 `gbrain jobs supervisor` 取代，后者处理脚本所做的一切，加上原子 PID 锁定、结构化审计事件、队列范围的健康检查和 SIGTERM 上的优雅排空。

**迁移：**

```bash
# 1. 停止并删除旧的 watchdog。
sudo kill $(head -n1 /tmp/gbrain-worker.pid) 2>/dev/null
sudo rm -f /usr/local/bin/minion-watchdog.sh /tmp/gbrain-worker.pid \
           /tmp/gbrain-worker.log
crontab -e   # 删除 "*/5 * * * * /usr/local/bin/minion-watchdog.sh" 行

# 2. 启动 supervisor（systemd 用户：从
#    docs/guides/minions-deployment-snippets/systemd.service 重新安装单元，
#    它现在调用 `gbrain jobs supervisor`）。
gbrain jobs supervisor start --detach --json
# 或者：sudo systemctl restart gbrain-worker

# 3. 验证。
gbrain jobs supervisor status --json
gbrain doctor   # 'supervisor' 检查应报告 running=true
```

### Schema / 迁移卫生

无论从哪个部署路径升级：

1. **升级前停止 worker。** `gbrain jobs supervisor stop`（或 `sudo systemctl stop gbrain-worker`）。跳过这一步可能导致进行中的任务遇到部分 schema。
2. **运行 `gbrain upgrade`**。然后如果 `gbrain doctor` 报告任何迁移为 `partial` 或 `pending`，运行 `gbrain apply-migrations --yes`。
3. **如果运行 shell 任务：** 从 v0.14 开始，向 supervisor 传递 `--allow-shell-jobs`（或在 `/etc/gbrain.env` 中保持 `GBRAIN_ALLOW_SHELL_JOBS=1`）。提交者不需要该标志；只有 worker 需要。
4. **验证。** `gbrain doctor` 应报告零 `pending` 或 `partial` 迁移，加上健康的 `supervisor` 检查。`gbrain jobs stats` 应显示升级前后 `dead` 没有无法解释的增长。

## 已知问题

### Supabase 连接断开

Worker 使用单个 Postgres 连接。如果 Supabase 断开它（维护、连接限制、网络中断），锁续约会无声失败。然后停顿检测器在 `max_stalled` 次错过后将任务死信。

**使问题更严重的当前默认值：**

- `lockDuration: 30000`（30秒）—— 对于连接中断期间的长任务来说太短。
- `max_stalled: 5`（schema 列默认值 —— 参见 `src/schema.sql` 和 `src/core/pglite-schema.ts`）。死信前五次心跳错过。
- `stalledInterval: 30000`（30秒）—— 检查过于激进。

**今天按任务调整。** `gbrain jobs submit` 接受 `--max-stalled N`、`--backoff-type fixed|exponential`、`--backoff-delay <ms>`、`--backoff-jitter 0..1` 和 `--timeout-ms N` 作为一流标志（从 v0.13.1 开始）。这些在提交时写入任务行 —— 这是 `handleStalled()` 读取的内容 —— 所以按任务调整是今天的真正旋钮。

### 不要向 `MinionWorker` 传递 `maxStalledCount`

这是一个无操作。停顿检测器读取行的 `max_stalled` 列（在提交时设置），而不是 `src/core/minions/worker.ts:74` 中的 worker 选项。改用 `gbrain jobs submit --max-stalled N` 按任务设置。

### Zombie shell 子进程

当 Bun worker 严重崩溃时，shell 任务的子进程可能变成僵尸。Supervisor 的 SIGTERM → 35秒排空 → SIGKILL 窗口覆盖 shell 处理器的 5 秒子进程终止宽限期（`KILL_GRACE_MS`）。对于长期运行的 shell 任务，建议通过提交时的 `--timeout-ms` 设置超时，而不是依赖硬终止。

## 冒烟测试

```bash
# Supervisor 存活？
gbrain jobs supervisor status --json | jq .running

# 聚合队列健康。
gbrain jobs stats

# 当前停滞的任务（仍为 `active`，lock_until 过期，重新入队前）。
gbrain jobs list --status active --limit 10

# 死信任务。
gbrain jobs list --status dead --limit 10

# Shell 处理器已注册？（检查 supervisor 审计日志或 worker stderr。）
gbrain jobs supervisor status --json | jq '.worker_config.allow_shell_jobs'
```

## 卸载

**`gbrain jobs supervisor`**（前台或 `--detach`）：

```bash
gbrain jobs supervisor stop
```

**systemd:**

```bash
sudo systemctl disable --now gbrain-worker
sudo rm /etc/systemd/system/gbrain-worker.service /etc/gbrain.env
sudo systemctl daemon-reload
```

**Fly / Render / Railway:** 从 `fly.toml` / `Procfile` 删除 `worker` 进程并重新部署。通过 `fly secrets` 设置的 secrets 会持续存在，直到 `fly secrets unset`。

**内联 `--follow`:** 删除 cron 条目。没有其他需要清理的 —— 临时 worker 随任务一起退出。