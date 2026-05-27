# 凭证网关（ClawVisor / Hermes）

让代理成为现实的三个集成。没有它们，大脑只是静态数据库。有了它们，大脑就活了。

### 14a. 凭证网关（ClawVisor / Hermes Gateway）

EA 工作流需要 Gmail、日历、联系人消息和消息传递访问权限。代理永远不应该直接持有 API 密钥。使用强制策略并在请求时注入凭证的凭证网关。

**OpenClaw: ClawVisor。** [ClawVisor](https://clawvisor.com) 是一个凭证保险库和授权网关，具有任务范围的授权。

**服务：** Gmail（列表、读取、发送、草稿）、Google 日历（CRUD）、Google Drive（列表、搜索、读取）、Google 联系人（列表、搜索）、Apple iMessage（列表、读取、搜索、发送）、GitHub、Slack。

**任务范围授权：** 每个请求必须包含来自已批准的常设任务的 `task_id`。任务声明：目的（详细，2-3 句话）、授权操作及预期使用模式、自动执行标志、生命周期（常设 vs 临时）。

**为什么这对 GBrain 重要：** EA 工作流需要 Gmail（分类前的发件人查找）、日历（会议准备、参会者页面）、联系人（丰富化触发）和 iMessage（直接指令）。ClawVisor 在不提供原始凭证的情况下为代理提供访问权限。

**设置：**

1. 在 ClawVisor 仪表板创建代理，复制代理令牌
2. 在环境中设置 `CLAWVISOR_URL` 和 `CLAWVISOR_AGENT_TOKEN`
3. 在仪表板中激活服务（Google、iMessage 等）
4. 创建具有广泛范围的常设任务（狭窄目的会导致错误阻止）
5. 将常设任务 ID 存储在代理内存中以供重用

**关键范围规则：** 在任务目的中要广泛。"完整的执行助理电子邮件管理，包括收件箱分类、按任何标准搜索、读取电子邮件、跟踪线程"有效。"电子邮件分类"会被拒绝。意图验证模型使用目的来判断每个请求是否一致 — 如果你的目的很窄，合法请求会验证失败。

**Hermes Agent: 内置网关。** Hermes 具有多平台消息传递（Telegram、Discord、Slack、WhatsApp、Signal、电子邮件）和内置到其网关的工具访问权限。使用 `config.yaml` 配置 API 凭证。网关守护进程管理连接并将 webhook 路由到代理会话。对于 Google 服务，在网关配置中配置 OAuth 凭证。Hermes 的计划自动化可以通过网关的工具系统运行相同的 EA 工作流（电子邮件分类、日历准备、联系人丰富化）。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。另见：[Getting Data In](README.md)*