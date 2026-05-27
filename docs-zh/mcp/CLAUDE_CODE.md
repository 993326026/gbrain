# 将 GBrain 连接到 Claude Code

## 选项 1：本地（推荐，无需服务器）

```bash
claude mcp add gbrain -- gbrain serve
```

就是这样。Claude Code 将 `gbrain serve` 作为 stdio 子进程生成。无需服务器、无需隧道、无需令牌。适用于 PGLite 和 Supabase 引擎。

## 选项 2：远程（从任何机器访问）

如果你在带有公共隧道的服务器上运行 GBrain（参见 [ngrok-tunnel recipe](../../recipes/ngrok-tunnel.md)）：

```bash
claude mcp add gbrain -t http \
  https://YOUR-DOMAIN.ngrok.app/mcp \
  -H "Authorization: Bearer YOUR_TOKEN"
```

将 `YOUR-DOMAIN` 替换为你的 ngrok 域名，`YOUR_TOKEN` 替换为来自 `gbrain auth create "claude-code"` 的令牌。

## 验证

在 Claude Code 中，尝试：

```
search for [any topic in your brain]
```

你应该看到来自 GBrain 知识库的结果。

## 移除

```bash
claude mcp remove gbrain
```