# 队列操作手册

"我的队列看起来卡住了 — 我该运行什么命令？" 下面的命令按你可能需要的顺序排列。v0.19.1 版本发布，此前发生了一次生产事件，队列在操作员注意到之前停滞了 90 多分钟。

## 第一个信号：任务没有运行

```bash
gbrain doctor --json | jq '.checks[] | select(.name == "queue_health")'
```

`queue_health` 标记两种模式：

- **stalled-forever**：`started_at` 早于 1 小时的活动任务。
- **waiting-depth**：任何按名称的队列深度超过 10（可通过 `GBRAIN_QUEUE_WAITING_THRESHOLD` 覆盖）。表示缺少 `maxWaiting`。

## 分类命令

```bash
# 当前谁在活跃？
gbrain jobs list --status active

# 谁在等待，按数量从多到少排序？
gbrain jobs list --status waiting --limit 50

# 某个特定任务有什么问题？
gbrain jobs get <id>
```

## 救援操作（按升级顺序）

```bash
# 强制终止单个卡住的任务：
gbrain jobs cancel <id>

# 完全清除特定任务（最后手段）：
gbrain jobs delete <id>

# 机制本身的健康检查：
gbrain jobs smoke --wedge-rescue
```

## 每个子检查的含义

- **stalled-forever** — 工作进程声明了一个任务，开始执行，并持有该行超过一小时。挂钟扫描会驱逐超过 `2× timeout_ms` 的任务；如果任务仍处于活动状态，要么没有设置 `timeout_ms`，要么扫描是新部署的，而此任务早于扫描部署。取消它。
- **waiting-depth** — 提交者提交任务的速度快于工作进程处理的速度。在提交时或程序化 `queue.add()` 调用上设置 `--max-waiting N`。如果想要更高的队列，通过 `GBRAIN_QUEUE_WAITING_THRESHOLD=50 gbrain doctor` 提高阈值。

## 自检：工作进程是否正在运行？

```bash
# 如果你运行 autopilot 时使用 --no-worker，请检查你的外部
# 工作进程（systemd / Docker / OpenClaw service-manager）是否存活：
gbrain jobs list --status active | head -5
```

如果列表为空且提交的任务不断堆积，说明没有工作进程在声明任务。启动一个：

```bash
GBRAIN_ALLOW_SHELL_JOBS=1 gbrain jobs work --concurrency 4
```

## v0.20+ 跟踪的后续工作

- B7 — `minion_workers` 心跳表用于真实活跃度检查（`--no-worker` 探测和已删除的 `queue_health` 工作进程心跳子检查都需要此功能）。
- B3 — `gbrain doctor --fix` 将学会救援队列卡住的情况。