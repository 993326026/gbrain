# 远程 MCP 部署选项

GBrain 的 MCP 服务器通过 `gbrain serve`（stdio 传输）运行。要使其可从其他设备和 AI 客户端访问，请在公共隧道后面运行 `gbrain serve --http`（内置 HTTP 传输，带 bearer 认证，仅支持 Postgres... 见 [DEPLOY.md](DEPLOY.md)）。以下是隧道选项。

## ngrok（推荐）

[ngrok](https://ngrok.com) 提供即时公共隧道。Hobby 级别（$8/月）提供永不改变的固定域名。

```bash
# 1. 安装 ngrok
brew install ngrok

# 2. 启动内置 HTTP 传输
gbrain serve --http --port 8787
# 有关令牌设置，请参阅 docs/mcp/DEPLOY.md

# 3. 通过 ngrok 暴露
ngrok http 8787 --url your-brain.ngrok.app
```

有关完整设置（包括认证令牌配置和固定域名设置），请参阅 [ngrok-tunnel recipe](../../recipes/ngrok-tunnel.md)。

## Tailscale Funnel

[Tailscale Funnel](https://tailscale.com/kb/1223/tailscale-funnel) 提供永久公共 HTTPS URL，带有自动 TLS。提供免费级别。最适合你控制两个端点的私有网络。

```bash
# 1. 安装 Tailscale
brew install tailscale

# 2. 暴露你的 MCP 服务器
tailscale funnel 8787
# 你的大脑现在位于 https://your-machine.ts.net
```

## Fly.io / Railway（始终在线）

对于需要 24/7 运行而不需要你的机器的生产部署：

- **Fly.io:** $5-10/月，全球边缘，`fly deploy`
- **Railway:** $5/月，git push 部署

两者都原生运行 Bun。无需打包，无需 Deno，无需冷启动，无超时限制。

## 比较

| | ngrok | Tailscale | Fly.io/Railway |
|--|---|---|---|
| 成本 | $8/月 (Hobby) | 免费 | $5-10/月 |
| 固定 URL | 是 (Hobby) | 是 | 是 |
| 笔记本电脑关闭时工作 | 否 | 否 | 是 |
| 冷启动 | 无 | 无 | 无 |
| 超时限制 | 无 | 无 | 无 |
| 全部 30 个操作 | 是 | 是 | 是 |
| 设置时间 | 5 分钟 | 10 分钟 | 15 分钟 |

**注意：** `gbrain serve --http` 是内置 HTTP 传输（v0.22.7+）。针对 `access_tokens` 表的 Bearer 认证、默认拒绝 CORS、双桶速率限制、主体限制、每请求审计日志。设计上仅支持 Postgres（PGLite 仅限本地）。有关环境变量和可调参数，请参阅 [DEPLOY.md](DEPLOY.md) 和 [SECURITY.md](../../SECURITY.md)。