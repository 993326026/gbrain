# 将 GBrain 连接到 Perplexity Computer

Perplexity Computer 支持带有 bearer 令牌认证的远程 MCP 服务器。

## 设置

1. 打开 Perplexity（需要 Pro 订阅）
2. 转到 **Settings > Connectors**（或 **MCP Servers**）
3. 添加新的远程连接器：
   - **URL:** `https://YOUR-DOMAIN.ngrok.app/mcp`
   - **Authentication:** API Key / Bearer Token
   - **Token:** 你的 GBrain 访问令牌（使用 `gbrain auth create "perplexity"` 创建）
4. 保存

将 `YOUR-DOMAIN` 替换为你的 ngrok 域名（设置参见 [ngrok-tunnel recipe](../../recipes/ngrok-tunnel.md)）。

## 验证

在 Perplexity 对话中，要求它使用你的大脑：

```
Use my GBrain to search for [topic]
```

## 注意

- Perplexity Computer 面向 Pro 订阅者提供
- Perplexity Mac 应用和 Web 版本都支持 MCP 连接器
- 如果你更喜欢 `gbrain serve`（stdio），Mac 应用也支持本地 MCP 服务器