# GBrain 部署拓扑

GBrain 支持三种部署形态。它们可以组合：单个用户可以在同一台机器上混合使用这三种，没有冲突，因为每种形态都解析为"当前哪个 `~/.gbrain/config.json` 是活动的？"，而 `GBRAIN_HOME` 控制该选择。

本文档涵盖三种拓扑、每种拓扑适合的场景以及具体的设置方案。将本文档与 `docs/architecture/brains-and-sources.md`（涵盖 brain 内组织轴）配对 —— 那篇文档是关于**哪个**数据库；本文档是关于**哪里**是数据库所在。

## 快速决策树

```
   "I'm setting up gbrain..."
        │
        ▼
  Just for me, on one machine? ─── yes ───▶ Topology 1 (single brain)
        │
        no
        │
        ▼
  Will a remote machine host the brain
  while my agent runs locally? ──── yes ───▶ Topology 2 (cross-machine thin client)
        │
        no
        │
        ▼
  Multiple Conductor worktrees that
  shouldn't share a code index? ─── yes ───▶ Topology 3 (split-engine)
```

拓扑 2 和 3 可以叠加：瘦客户端安装也可以托管每个工作树的代码引擎，每个工作树的代码引擎也可以将其 artifact brain 指向远程服务器。

## Topology 1 — Single brain（当前默认）

```
  ┌────────────────┐
  │   one machine  │
  │  ┌──────────┐  │
  │  │  gbrain  │──┼──→  ~/.gbrain/  →  PGLite  or  Supabase
  │  │   CLI    │  │
  │  └──────────┘  │
  └────────────────┘
```

你得到的：一个本地数据库（小型 brain 使用 PGLite，~1000+ 文件使用 Supabase）。所有命令直接对其工作。`gbrain serve` 通过 MCP 向单个代理暴露它。

适合场景：单人使用、单机器、一个代理、没有 Conductor 并行性。这是默认设置；`gbrain init`（无标志）为你提供这个。

设置：

```
gbrain init           # 交互式 —— 默认 PGLite
gbrain init --pglite  # 显式本地
gbrain init --supabase  # 远程 Supabase（推荐用于 1000+ 文件）
```

这里没有其他特别之处。其他两种拓扑是"谁拥有数据库"和"代理如何与它通信"的变体。

## Topology 2 — Cross-machine thin client

```
  ┌────────────┐                    ┌──────────────────┐
  │ neuromancer│                    │    brain-host    │
  │ ┌────────┐ │ HTTP MCP / OAuth   │  ┌────────────┐  │
  │ │ Hermes │─┼───────────────────→│  │   gbrain   │──┼──→ Supabase
  │ │ agent  │ │                    │  │ serve --http│  │
  │ └────────┘ │                    │  └────────────┘  │
  │            │                    │   (with autopilot)│
  │  no local  │                    │                  │
  │  gbrain DB │                    │                  │
  └────────────┘                    └──────────────────┘
```

你得到的：一台机器上的代理（"neuromancer"）通过 HTTP MCP 和 OAuth 使用托管在另一台机器（"brain-host"）上的 brain。代理的机器没有本地引擎。所有查询、搜索、嵌入和索引都在主机上进行。

适合场景：

- 重型 brain（Supabase + autopilot）驻留在一台强大的机器上；其他地方的代理只是消费它。
- 你希望在多台机器之间有一个真相来源。
- 启动并行本地安装会造成源 ID 冲突或重复工作。

瘦客户端的 `~/.gbrain/config.json` 带有 `remote_mcp` 字段而不是本地数据库连接：

```jsonc
{
  "engine": "postgres",  // 忽略 —— 从未使用
  "remote_mcp": {
    "issuer_url": "https://brain-host.local:3001",
    "mcp_url":    "https://brain-host.local:3001/mcp",
    "oauth_client_id": "neuromancer-...",
    "oauth_client_secret": "..."  // 或设置 GBRAIN_REMOTE_CLIENT_SECRET
  }
}
```

CLI 调度守卫拒绝任何数据库绑定命令（`sync`、`embed`、`extract`、`migrate`、`apply-migrations`、`repair-jsonb`、`orphans`、`integrity`、`serve`）在瘦客户端安装上运行，并给出指向远程主机的清晰错误。`gbrain doctor` 运行专门的瘦客户端检查集（OAuth 发现、令牌往返、MCP 烟雾测试）。

### 设置

**步骤 1 — 在主机上（brain-host）：**

```bash
gbrain init --supabase                         # 或 --pglite，无所谓
gbrain serve --http --port 3001                # 暴露 /mcp + OAuth
gbrain auth register-client neuromancer \
  --grant-types client_credentials \
  --scopes read,write,admin                    # admin 需要用于 ping/doctor
```

`register-client` 命令打印 `client_id` 和 `client_secret`。记下两者。**范围必须包含 `admin`** —— `submit_job`（`gbrain remote ping` 使用）和 `run_doctor`（`gbrain remote doctor` 使用）都需要它。

**步骤 2 — 在瘦客户端上（neuromancer）：**

```bash
gbrain init --mcp-only \
  --issuer-url https://brain-host.local:3001 \
  --mcp-url https://brain-host.local:3001/mcp \
  --oauth-client-id <id> \
  --oauth-client-secret <secret>
```

飞行前烟雾测试运行三个探测（OAuth 发现、令牌往返、MCP 初始化）。如果任何一个失败，init 会以可操作的错误退出。成功后，`~/.gbrain/config.json` 设置 `remote_mcp`，不创建本地数据库。

**步骤 3 — 配置代理的 MCP 客户端。**

对于 Claude Desktop / Hermes / openclaw，添加一个指向主机 `mcp_url` 的 MCP 服务器条目，并使用 `register-client` 提供的 bearer 令牌。Claude Desktop 的 `~/.config/claude/claude_desktop_config.json` 示例：

```jsonc
{
  "mcpServers": {
    "gbrain": {
      "type": "url",
      "url": "https://brain-host.local:3001/mcp",
      "headers": { "Authorization": "Bearer <client_secret>" }
    }
  }
}
```

**步骤 4 — 验证。**

```bash
gbrain doctor             # 运行瘦客户端检查（不需要本地数据库）
gbrain remote ping        # 在主机上触发 autopilot 周期（Tier B）
gbrain remote doctor      # 请求主机运行自己的 doctor（Tier B）
```

`gbrain sync` 及其同类命令会以明确的瘦客户端错误拒绝，命名 `mcp_url`。这是正确的行为 —— 这些命令需要本地引擎，这里不存在。

### 重新运行守卫

在已设置瘦客户端配置的机器上运行 `gbrain init`（无标志）会在没有 `--force` 的情况下拒绝。这捕获了编排器不断尝试创建本地数据库的脚本设置循环摩擦。使用 `gbrain init --mcp-only --force` 刷新瘦客户端配置。

### 存储 OAuth 密钥

优先级顺序中的三个存储路径：

1. **`GBRAIN_REMOTE_CLIENT_SECRET` 环境变量**（无头代理首选）。设置后，覆盖配置文件中的任何内容。当环境变量是源时，初始化流程不会持久化配置文件副本。
2. **`~/.gbrain/config.json` 具有 0600 权限**（交互式设置的默认值；镜像 Supabase 密钥当前的存储方式）。
3. macOS Keychain 集成在路线图上；不在 v1 中。

## Topology 3 — Split-engine, per-worktree code + remote artifacts

```
  ┌──────────────────────────────────────────────────────┐
  │                  one machine                         │
  │                                                      │
  │  ┌─ worktree A ──────────────┐                       │
  │  │  GBRAIN_HOME=A/.conductor │                       │
  │  │  gbrain serve --port 3001 │── PGLite (code A)     │
  │  └───────────────────────────┘                       │
  │                                                      │
  │  ┌─ worktree B ──────────────┐                       │
  │  │  GBRAIN_HOME=B/.conductor │                       │
  │  │  gbrain serve --port 3002 │── PGLite (code B)     │
  │  └───────────────────────────┘                       │
  │                                                      │
  │  ┌─ default ~/.gbrain ───────┐    HTTP MCP / OAuth   │
  │  │  gbrain serve --port 3000 │──────────────────────→ remote artifacts
  │  └───────────────────────────┘                        (Supabase / brain-host)
  │                                                      │
  │  Agent's MCP config (Hermes / Claude Desktop):       │
  │    mcp__gbrain_code__*       → http://localhost:3001 │
  │    mcp__gbrain_artifacts__*  → http://brain-host/mcp │
  └──────────────────────────────────────────────────────┘
```

你得到的：每个 Conductor worktree 都有自己的每个 worktree 代码索引（本地 PGLite，worktree 消亡时可丢弃）。Artifacts（计划、学习、转录）仍然存在于所有 worktree 都可以看到和写入的共享 brain 中。

适合场景：

- 一台机器上有多个 Conductor worktree，都接触同一个代码仓库。
- 您不希望每个 worktree 的代码导入覆盖其他 worktree 的 `last_commit`、源 ID 或符号表。
- 您**确实**希望 artifacts（计划、学习、回顾、转录）在 worktree 之间可见。

### 工作原理

`GBRAIN_HOME` 选择哪个 `~/.gbrain` 目录是活动的。每个 worktree 设置：

```bash
export GBRAIN_HOME=/path/to/worktree-A/.conductor/gbrain
gbrain init --pglite
gbrain serve --http --port 3001
```

每个 worktree 的 `gbrain serve` 实例绑定自己的端口并索引自己的数据库。多个 `gbrain serve` 进程可以很好地共存 —— 它们是具有独立配置和独立连接池的独立 OS 进程。

Artifact brain 作为单独的 `gbrain serve` 实例运行，使用默认的 `~/.gbrain`（无 GBRAIN_HOME 覆盖）—— 或者远程运行，在这种情况下它是 Topology 2 设置。

代理的 MCP 客户端配置列出多个服务器，每个服务器都有唯一的别名。工具名称命名空间为 `mcp__<alias>__<tool>`，因此代理调用 `mcp__gbrain_code__search` 进行代码查找，调用 `mcp__gbrain_artifacts__search` 进行 artifact 查找。

### 关键：别名级路由是手动的

拓扑 3 在 gbrain 内部没有智能的每个工具路由。代理在选择别名时选择查询哪个 brain。**错误的别名会静默地写入（或查询）错误的 brain。** 这是有意的（显式胜过魔法）但真实存在：

- 如果代理用代码形状的内容调用 `mcp__gbrain_artifacts__put_page`，该页面会永远留在 artifact brain 中。
- 如果代理为实际上需要 artifact 上下文的问题调用 `mcp__gbrain_code__search`，搜索会返回空。

缓解措施：

- 清晰地命名别名。`gbrain_code` vs `gbrain_artifacts` 是明确的；`gbrain` vs `gbrain_local` 不是。
- 在代理的系统提示或规则中记录哪个别名去哪里。明确说明"代码问题 → `gbrain_code`；其他一切 → `gbrain_artifacts`。"
- 将拓扑 3 与 gstack 的每个 worktree 接线（在 worktree 之间一致地设置别名名称 + 代理规则）配对。

### 设置（手动；gstack 自动化这方面）

gbrain 端不需要新代码 —— `GBRAIN_HOME` 和 `--port` 已经存在。设置看起来像：

```bash
# 在端口 3000 上启动 artifact brain（默认 ~/.gbrain）
gbrain serve --http --port 3000 &

# 在端口 3001 上启动每个 worktree 的代码 brain
export GBRAIN_HOME=/path/to/worktree-A/.conductor/gbrain
gbrain init --pglite
gbrain serve --http --port 3001 &
unset GBRAIN_HOME
```

然后用两个条目（不同的别名、不同的端口）配置代理的 MCP 配置。对于 Claude Desktop：

```jsonc
{
  "mcpServers": {
    "gbrain_artifacts": {
      "type": "url",
      "url": "http://localhost:3000/mcp",
      "headers": { "Authorization": "Bearer <token-A>" }
    },
    "gbrain_code": {
      "type": "url",
      "url": "http://localhost:3001/mcp",
      "headers": { "Authorization": "Bearer <token-B>" }
    }
  }
}
```

gstack 端的接线（每个 worktree 的 home 设置、端口分配、自动 MCP 配置生成、每个 worktree 数据库的 gitignore）在 gstack repo 的 setup-gbrain skill 中 —— 它组合这些原语，gbrain 不需要知道 Conductor。

## 组合拓扑

三种形态可以组合。一台机器可以运行：

- 指向远程 artifact brain 的瘦客户端默认配置（拓扑 2）。
- 加上在自己的 `GBRAIN_HOME` 下的每个 worktree 代码 brain（拓扑 3）。
- 每个 worktree 的 `gbrain serve` 实例是本地的；代理的 MCP 配置将它们与远程 artifact brain 一起列出。

`GBRAIN_HOME` 控制任何一次 CLI 调用哪个配置文件是活动的。`gbrain serve --port` 控制服务器监听哪个端口。代理的 MCP 客户端选择别名，从而选择每个工具调用的目标。没有全局 gbrain 编排器同时知道所有这些 —— 这是设计使然。

## 何时不使用这些拓扑

- **如果您的代理只在与 brain 相同的机器上运行，请不要使用拓扑 2。** 本地 `gbrain` 安装 + `gbrain serve`（stdio）更简单更快。
- **如果您一次只有一个 Conductor worktree，请不要使用拓扑 3。** 每个 worktree 的引擎存在是为了防止冲突；一次使用一个不会有冲突。
- **不要在同一台机器上的同一个 `GBRAIN_HOME` 中同时使用 `remote_mcp` 瘦客户端和本地引擎。** 当设置了 `remote_mcp` 时，调度守卫拒绝数据库绑定命令。如果您确实想在一台机器上同时使用两种模式，请使用 `GBRAIN_HOME` 分隔它们（一个 home 用于瘦客户端，另一个用于本地引擎）。

## 另见

- `docs/architecture/brains-and-sources.md` — brain 内组织（brains vs sources 轴）。
- `docs/mcp/CLAUDE_DESKTOP.md` 和同类文档 — 每个客户端的 MCP 设置。
- `gbrain init --help` 和 `gbrain auth --help` 获取命令级详细信息。