# Minions 修复 —— 修复半迁移安装

**简而言之：** 在 v0.11.1+ 版本上，一切应该自我修复。如果 Minions 部分设置（没有 `~/.gbrain/preferences.json`，autopilot 仍然内联运行，cron 任务仍然调用 `agentTurn`），运行：

```bash
gbrain apply-migrations --yes
```

它是幂等的。在已经迁移的 v0.11.1 安装上，这是一个廉价的空操作。

## 背景

v0.11.0 发布了 Minions schema、队列、worker 和迁移技能 —— 但迁移技能本身在升级时从未触发。`runPostUpgrade` 打印了功能介绍后就停止了。v0.11.0 从未公开发布；v0.11.1 是第一个公开发布的 Minions 版本，并修复了这个重大 bug（迁移在 `gbrain upgrade` 和 `postinstall` 钩子上自动触发）。

如果你在 v0.11.1 之前的分支构建上（例如，在 v0.11.1 标记之前运行 `minions-jobs` 分支），Minions 可能已安装但未连接：schema 是 v7，但没有 `~/.gbrain/preferences.json`，autopilot 仍然内联运行，cron 任务仍然调用 `agentTurn`。

本指南涵盖两条路径：标准的 v0.11.1+ 修复，以及没有 `apply-migrations` 的 v0.11.1 之前版本的临时解决方案。

## 检测半迁移状态

```bash
gbrain doctor
```

如果安装是半迁移状态，你会看到：

```
[FAIL] minions_migration: MINIONS HALF-INSTALLED (partial migration: 0.11.0). Run: gbrain apply-migrations --yes
```

或者

```
[FAIL] minions_config: MINIONS HALF-INSTALLED (schema v7+ but no ~/.gbrain/preferences.json). Run: gbrain apply-migrations --yes
```

对于机器可读报告（适合 cron）：

```bash
gbrain skillpack-check --quiet && echo healthy || echo needs_action
gbrain skillpack-check | jq -r '.actions[]'    # 打印要运行的确切命令
```

## 修复（v0.11.1 或更高版本）

```bash
gbrain apply-migrations --yes
```

读取 `~/.gbrain/migrations/completed.jsonl`，与 TS 迁移注册表比较差异，运行任何待处理的迁移。七个阶段：

```
A. Schema        gbrain init --migrate-only
B. Smoke         gbrain jobs smoke
C. Mode          prompt (或 --yes 默认 pain_triggered)
D. Prefs         写入 ~/.gbrain/preferences.json
E. Host          AGENTS.md 标记注入 + cron 重写为 gbrain
                 内置命令；特定主机处理程序的 JSONL TODO
F. Install       gbrain autopilot --install (环境感知)
G. Record        追加 completed.jsonl status:"complete"
```

如果阶段 E 为特定主机处理程序发出 TODO（例如，你的 OpenClaw 的 ~29 个非 gbrain cron），迁移以 `status: "partial"` 完成。你的主机代理使用 `skills/migrations/v0.11.0.md` + `docs/guides/plugin-handlers.md` 处理 TODO，在主机仓库中发布处理程序注册，然后重新运行 `gbrain apply-migrations --yes`。新注册的 cron 条目被重写，JSONL 行标记 `status: "complete"`。

## 临时解决方案（v0.11.1 之前的版本，还没有 apply-migrations）

如果你卡在没有 `apply-migrations` 的分支构建上：

```bash
curl -fsSL https://raw.githubusercontent.com/garrytan/gbrain/v0.11.1/scripts/fix-v0.11.0.sh | bash
```

这个 bash 脚本从 shell 环境执行 apply-migrations 的功能：

1. `gbrain init --migrate-only` — schema v7。
2. `gbrain jobs smoke` — 验证 Minions 健康状况。
3. 提示输入 `minion_mode`（非 TTY 上默认为 `pain_triggered`）。
4. 原子写入 `~/.gbrain/preferences.json`。
5. 向 `~/.gbrain/migrations/completed.jsonl` 追加 `status: "partial"` 和 `apply_migrations_pending: true`。这个部分记录是 v0.11.1 的 `apply-migrations` 在用户升级后继续完成剩余阶段的信号。
6. 检测主机代理仓库并打印重写说明（从不从 curl 管道脚本自动编辑）。
7. 打印下一步：`Run: gbrain autopilot --install`。

安装 v0.11.1 后，重新运行 `gbrain apply-migrations --yes` 以完成剩余阶段（主机重写 + autopilot 安装）。临时解决方案的 `status: "partial"` 记录设计为干净恢复（不会破坏永久迁移路径）。

## 验证修复是否生效

```bash
# 1. 偏好设置存在且可读
cat ~/.gbrain/preferences.json

# 2. 迁移已记录
cat ~/.gbrain/migrations/completed.jsonl

# 3. Autopilot 正在监控 Minions worker 子进程
gbrain autopilot --status
ps aux | grep 'jobs work'

# 4. 任务显示在队列中
gbrain jobs list

# 5. 任何特定主机的待处理 TODO
cat ~/.gbrain/migrations/pending-host-work.jsonl 2>/dev/null || echo "(none — all host work is done)"

# 6. Doctor + skillpack-check 都应该干净
gbrain doctor
gbrain skillpack-check --quiet && echo ok
```

## 如果修复失败

每个阶段都是幂等的。重新运行是安全的。常见失败模式：

- **阶段 B smoke 失败：** schema 没有应用。检查 `~/.gbrain/config.json` 是否有有效的 `database_url`（或 PGLite 的 `database_path`）。直接运行 `gbrain init --migrate-only` 并查看错误。
- **阶段 F install 失败：** 你的主机环境不匹配任何检测到的目标。显式传递 `--target <macos|linux-systemd|ephemeral-container|linux-cron>`。
- **待处理的主机工作从未清除：** 你的主机代理尚未发布处理程序注册。阅读 `~/.gbrain/migrations/pending-host-work.jsonl`，打开 `skills/migrations/v0.11.0.md`，并按照主机代理说明手册操作。

## 相关文档

- `skills/migrations/v0.11.0.md` — 主机代理的完整迁移技能。
- `skills/skillpack-check/SKILL.md` — 何时以及如何运行健康检查。
- `docs/guides/plugin-handlers.md` — 特定主机处理程序的插件契约。
- `skills/conventions/cron-via-minions.md` — 标准 cron 重写模式。