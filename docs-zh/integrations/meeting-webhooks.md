# 会议和通话 Webhook

### 14b. Circleback -- 通过 Webhook 摄入会议

[Circleback](https://circleback.ai) 录制会议，生成带说话人分割的记录，并在完成时触发 webhook。

**Webhook 设置：**

1. 在 Circleback 仪表板 -> Automations -> 添加 webhook
2. URL: `{your_agent_gateway}/hooks/circleback-meetings`
3. Circleback 提供签名密钥用于 HMAC-SHA256 签名验证
4. 将签名密钥存储在你的 webhook 转换器中用于验证

**Webhook 负载：** 包含 id、名称、参会者、笔记、行动项、完整记录、日历事件上下文的会议 JSON。

**签名验证：** 头部 `X-Circleback-Signature` 包含 `sha256=<hex>`。使用 `HMAC-SHA256(body, signing_secret)` 验证。拒绝未验证的 webhook。

**API 访问的 OAuth：** Circleback 使用动态客户端注册（OAuth 2.0）。访问令牌约 24 小时过期，通过刷新令牌自动刷新。将凭证存储在代理内存中。

**流程：** Webhook 触发 -> 转换器验证签名 + 规范化 -> 代理唤醒 -> 通过 API 拉取完整记录 -> 创建大脑会议页面 -> 传播到实体页面 -> 提交到大脑仓库 -> `gbrain sync`。

### 14c. Quo (OpenPhone) -- SMS 和通话集成

[Quo](https://openphone.com)（前身为 OpenPhone）提供带有 SMS、通话、语音信箱和 AI 记录的商务电话号码。

**Webhook 设置：**

1. 在 Quo 仪表板 -> Integrations -> Webhooks
2. 注册 webhook 用于：`message.received`、`call.completed`、`call.summary.completed`、`call.transcript.completed`
3. 全部指向：`{your_agent_gateway}/hooks/quo-events`
4. 将注册的 webhook ID 存储在代理内存中

**入站短信工作原理：**

- Webhook 触发，包含发件人电话、消息文本、对话上下文
- 代理通过电话号码在大脑中查找发件人
- 向用户的消息平台显示发件人身份 + 大脑上下文
- 起草回复供批准（未经明确许可永不自动回复）

**入站通话工作原理：**

- `call.completed` 触发 -> 如果时长 > 30 秒，通过 API 获取记录 + AI 摘要
- 摄入到大脑（会议风格页面位于 `meetings/`）
- 更新相关人员和公司页面

**API 认证：** `Authorization` 头部中的裸 API 密钥（无 Bearer 前缀）。

**关键端点：** `POST /v1/messages`（发送 SMS）、`GET /v1/messages`（列表）、`GET /v1/call-transcripts/{id}`、`GET /v1/conversations`。

---

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。另见：[Getting Data In](README.md)*