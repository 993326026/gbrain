# 部署 GBrain 远程 MCP 服务器

> **v0.26.0+:** `gbrain serve --http` 提供完整的 OAuth 2.1（客户端凭证、授权码 + PKCE、刷新轮换、可选 DCR）、在 `/admin` 的嵌入式 React 管理仪表板、作用域操作和实时 SSE 活动 feed。v0.26 之前的旧版 bearer 令牌仍然有效 — `verifyAccessToken` 回退到 `access_tokens` 表并将令牌继承为 `read+write+admin`。旧版回退仅支持 Postgres（`access_tokens` 表仅存在于 Postgres）；OAuth 表在 PGLite 和 Postgres 上都能工作。有关环境变量和可调默认值，请参阅 [SECURITY.md](../../SECURITY.md)。

从任何设备、任何 AI 客户端访问你的大脑。GBrain 提供两种传输方式：`gbrain serve`（stdio）用于本地代理，`gbrain serve --http`（v0.26.0+）用于通过 OAuth 2.1 的远程客户端。

## 三种路径

### 本地 stdio（零设置）

```bash
gbrain serve
```

适用于 Claude Code、Cursor、Windsurf 和任何支持 stdio 的 MCP 客户端。无需服务器、无需隧道、无需令牌。适用于 PGLite 和 Postgres 引擎。

### 通过 OAuth 2.1 远程（推荐，v0.26.0+）

```bash
gbrain serve --http --port 3131
ngrok http 3131 --url your-brain.ngrok.app
gbrain serve --http --port 3131 --public-url https://your-brain.ngrok.app
```

内置 HTTP 传输，带有 OAuth 2.1、作用域操作、在 `/admin` 的管理仪表板和实时 SSE 活动 feed。零外部依赖。这是唯一适用于 ChatGPT 的路径（ChatGPT MCP 连接器要求 OAuth 2.1 + PKCE）。当服务器可以通过 `http://localhost:<port>` 以外的地址访问时，传递 `--public-url`，以便发现元数据中的 OAuth issuer 与客户端访问的地址匹配（RFC 8414 §3.3）。

支持的客户端：
- **ChatGPT** — 需要 OAuth 2.1 + PKCE。与 `--http` 原生配合使用。
- **Claude Desktop / Cowork** — OAuth 2.1 或旧版 bearer 令牌。
- **Perplexity** — OAuth 2.1 客户端凭证授权。
- **Claude Code、Cursor、Windsurf** — 可以使用 OAuth 或旧版 bearer。

请参阅下面的 [OAuth 2.1 设置](#oauth-21-setup-v100) 部分。

### 使用旧版 bearer 令牌远程（v0.26 之前的部署）— 仅 Postgres

```
Your AI client (Claude Desktop, Perplexity, etc.)
  → ngrok tunnel (https://YOUR-DOMAIN.ngrok.app)
  → gbrain serve --http  (内置传输，带 bearer 认证)
  → Postgres (连接池或自托管)
```

这需要：
1. Postgres 支持的大脑（`access_tokens` 表仅存在于 Postgres；对 PGLite 安装运行 `gbrain serve --http` 会在启动时快速失败）
2. 运行 `gbrain serve --http` 的机器
3. 公共隧道（ngrok、Tailscale 或云主机）
4. 通过 `gbrain auth create <name>` 创建的 bearer 令牌

升级到 HTTP 服务器时，v1.0 之前的令牌继承为 `read+write+admin` 作用域，因此不需要迁移。

## OAuth 2.1 设置（v0.26.0+）

### 1. 启动 HTTP 服务器

```bash
gbrain serve --http --port 3131
```

首次启动时，服务器向 stderr 打印 **admin bootstrap token**：

```
Admin bootstrap token: 3a1f9c...
Open http://localhost:3131/admin and paste it to log in.
```

保存此令牌。打开 `http://localhost:3131/admin` 并粘贴以访问仪表板。仪表板显示实时活动、已注册客户端、请求日志和每客户端配置导出。

> **v0.26.9+:** `mcp_request_log.params` 和实时 SSE 活动 feed 默认显示脱敏摘要 `{redacted, kind, declared_keys, unknown_key_count, approx_bytes}`。声明的参数键被保留（与操作的规范交叉）；未知键被计数但从不命名，字节大小向上舍入到 1KB，因此大小探测攻击无法二分搜索秘密内容。个人笔记本电脑上的操作员如果想要原始有效负载，可以传递 `gbrain serve --http --log-full-params`（启动时会发出警告）。多租户部署应保持脱敏默认值。

### 2. 注册 OAuth 客户端

从 **`/admin` 仪表板**注册客户端：

1. 点击 **Register client**。
2. 输入名称（例如 `perplexity`、`chatgpt`）。
3. 选择作用域：`read`、`write`、`admin`（复选框）。
4. 选择授权类型：`client_credentials` 用于机器对机器（Perplexity、Claude Desktop bearer 模式）或 `authorization_code` 用于带 PKCE 的基于浏览器的客户端（ChatGPT）。
5. 对于 `authorization_code` 客户端，粘贴重定向 URI。
6. 点击 **Register**。凭证显示模态框显示 `client_id`（以及机密客户端的 `client_secret`）一次。立即复制或下载 JSON — 秘密在存储时被哈希处理，不再显示。

或者从 CLI — 脚本化更快：

```bash
gbrain auth register-client perplexity \
  --grant-types client_credentials \
  --scopes "read write"
```

主机仓库包装器可以编程方式注册：

```ts
await oauthProvider.registerClientManual(
  'perplexity',
  ['client_credentials'],
  'read write',
  [],  // redirect_uris, CC 为空
);
```

对于自助服务客户端注册（动态客户端注册，RFC 7591），使用 `--enable-dcr` 启动服务器。DCR 默认关闭。

### 3. 暴露服务器

```bash
brew install ngrok
ngrok config add-authtoken YOUR_TOKEN
ngrok http 3131 --url your-brain.ngrok.app
```

你的 OAuth issuer URL 变为 `https://your-brain.ngrok.app`。MCP SDK 的路由器在 `/.well-known/oauth-authorization-server` 暴露符合规范的发现端点。

### 4. 作用域和 localOnly

每个操作都标记为 `read | write | admin`。四个操作是 `localOnly`，无论作用域如何，都会通过 HTTP 拒绝：`sync_brain`、`file_upload`、`file_list`、`file_url`。远程代理无法访问本地文件系统。

| 作用域 | 允许的操作 |
|--------|-----------|
| `read` | `search`、`query`、`get_page`、`list_pages`、图遍历 |
| `write` | `put_page`、`delete_page`、`add_link`、`add_timeline_entry` |
| `admin` | 客户端管理、令牌撤销、扫描、本地操作 |

## 旧版 Bearer 令牌设置

如果你尚未准备好迁移，请继续使用 v0.26 之前的 bearer 令牌。它们在 HTTP 服务器上继承为 `read+write+admin` 作用域。

### 1. 设置隧道

完整设置请参阅 [ngrok-tunnel recipe](../../recipes/ngrok-tunnel.md)。快速版本：

```bash
brew install ngrok
ngrok config add-authtoken YOUR_TOKEN
ngrok http 8787 --url your-brain.ngrok.app  # Hobby tier for fixed domain
```

### 2. 创建访问令牌

```bash
# 为每个客户端创建一个令牌
gbrain auth create "claude-desktop"

# 列出所有令牌
gbrain auth list

# 撤销令牌
gbrain auth revoke "claude-desktop"
```

令牌是每个客户端的。为每个设备/应用创建一个。如果泄露，单独撤销。令牌以 SHA-256 哈希形式存储在你的数据库中。

### 3. 连接你的 AI 客户端

- **ChatGPT:** [设置指南](CHATGPT.md)（OAuth 2.1 + PKCE，需要 `gbrain serve --http`）
- **Claude Code:** [设置指南](CLAUDE_CODE.md)
- **Claude Desktop:** [设置指南](CLAUDE_DESKTOP.md)（必须使用 GUI，不是 JSON 配置）
- **Claude Cowork:** [设置指南](CLAUDE_COWORK.md)
- **Perplexity:** [设置指南](PERPLEXITY.md)

### 4. 验证

```bash
gbrain auth test \
  https://YOUR-DOMAIN.ngrok.app/mcp \
  --token YOUR_TOKEN
```

## 操作

所有 30 个 GBrain 操作都可远程使用，包括 `sync_brain` 和 `file_upload`（自托管服务器无超时限制）。

**`file_upload` 安全注意事项：** 远程 MCP 调用者被限制在启动 `gbrain serve` 的工作目录中。符号链接、`..` 遍历和 cwd 之外的绝对路径被拒绝。页面 slug 和文件名经过白名单验证（字母数字 + 连字符；无控制字符、RTL 覆盖或反斜杠）。本地 CLI 调用者（`gbrain file upload ...`）保持不受限制的文件系统访问，因为用户拥有机器。

## 部署选项

有关 ngrok、Tailscale Funnel 和云主机（Fly.io、Railway）的比较，请参阅 [ALTERNATIVES.md](ALTERNATIVES.md)。

## 故障排除

**"missing_auth" 错误**
包含 Authorization 头部：`Authorization: Bearer YOUR_TOKEN`

**"invalid_token" 错误**
运行 `gbrain auth list` 查看活动令牌。

**"service_unavailable" 错误**
数据库连接失败。检查你的 Supabase 仪表板是否有中断。

**Claude Desktop 无法连接**
远程服务器必须通过 Settings > Integrations 添加，**不是** `claude_desktop_config.json`。参见 [CLAUDE_DESKTOP.md](CLAUDE_DESKTOP.md)。

## 预期延迟

| 操作 | 典型延迟 | 备注 |
|------|----------|------|
| get_page | < 100ms | 单个数据库查询 |
| list_pages | < 200ms | 带过滤器的数据库查询 |
| search (keyword) | 100-300ms | 全文搜索 |
| query (hybrid) | 1-3s | 嵌入 + 向量 + 关键词 + RRF |
| put_page | 100-500ms | 写入 + 触发 search_vector 更新 |
| get_stats | < 100ms | 聚合查询 |

**注意：** `gbrain serve --http` 在 v0.26.0 中发布，OAuth 2.1 + 管理仪表板内置到二进制文件中。自定义 HTTP 包装模式（参见 [voice recipe](../../recipes/twilio-voice-brain.md)）对于需要定制中间件的团队仍然支持，但对于大多数远程部署，内置服务器是推荐路径。