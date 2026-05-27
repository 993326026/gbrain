# 将 GBrain 连接到 Claude Cowork

有两种方法将 GBrain 接入 Cowork 会话：

## 选项 1：远程（通过自托管服务器 + 隧道）

对于团队/企业计划，组织所有者添加连接器：

1. 转到 **Organization Settings > Connectors**
2. 使用 MCP 服务器 URL 添加新连接器：
   ```
   https://YOUR-DOMAIN.ngrok.app/mcp
   ```
3. 在 Advanced Settings 中添加 Bearer 令牌认证（使用 `gbrain auth create "cowork"` 创建一个）
4. 保存

注意：Cowork 从 Anthropic 的云连接，而不是你的设备。你的服务器必须可公开访问（ngrok、Tailscale Funnel 或云托管）。

## 选项 2：本地桥接（通过 Claude Desktop）

如果你已经在 Claude Desktop 中配置了 GBrain（通过 `gbrain serve` stdio 或远程集成），Cowork 会自动获得访问权限。Claude Desktop 通过其 SDK 层将本地 MCP 服务器桥接到 Cowork。

这意味着：如果 `gbrain serve` 正在运行并在 Claude Desktop 中配置，你不需要为 Cowork 设置单独的服务器。

## 选择哪一个？

- **远程服务器：** 即使你的笔记本电脑关闭也能工作，对所有组织成员可用
- **本地桥接：** 如果 Claude Desktop 已经有 GBrain，则无需额外设置，但需要你的机器运行