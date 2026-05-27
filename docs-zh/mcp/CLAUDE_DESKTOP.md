# 将 GBrain 连接到 Claude Desktop

**重要：** Claude Desktop **不**通过 `claude_desktop_config.json` 连接到远程 MCP 服务器。该文件仅适用于本地 stdio 服务器。远程 HTTP 服务器必须通过 GUI 添加。

## 设置

1. 打开 Claude Desktop
2. 转到 **Settings > Integrations**
3. 点击 **Add Integration**（或 **Add Connector**）
4. 输入 MCP 服务器 URL：
   ```
   https://YOUR-DOMAIN.ngrok.app/mcp
   ```
   将 `YOUR-DOMAIN` 替换为你的 ngrok 域名（设置参见 [ngrok-tunnel recipe](../../recipes/ngrok-tunnel.md)）。
5. 将认证设置为 **Bearer Token** 并粘贴你的令牌（使用 `gbrain auth create "claude-desktop"` 创建）
6. 保存

## 验证

开始新对话并尝试：

```
Search my brain for [any topic]
```

Claude Desktop 将自动使用你的 GBrain 工具。

## 常见错误

**对远程服务器使用 claude_desktop_config.json** — 这会静默失败，没有错误消息。JSON 配置仅适用于本地 stdio MCP 服务器。远程 HTTP 服务器必须通过 GUI 中的 Settings > Integrations 添加。

**使用错误的 URL** — 确保 URL 以 `/mcp` 结尾（不是 `/health` 或只是基础域名）。