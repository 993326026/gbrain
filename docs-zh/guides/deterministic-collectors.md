# 确定性收集器：代码处理数据，LLM 处理判断

## 目标

将机械工作（100% 可靠的代码）与分析工作（LLM 判断）分离，确保确定性任务永远不会概率性失败。

## 用户获得的价值

没有这个：LLM 生成 Gmail 链接、格式化表格、跟踪状态。前 10 个条目遵循规则，第 11 个条目就漏掉了链接。你在提示中写"NO EXCEPTIONS"，它仍然失败。20 个条目的 90% 可靠性意味着每天可见失败两次。信任被摧毁。

有了这个：代码处理 URL、格式化和状态（100% 可靠）。LLM 读取预格式化的数据并添加判断、分类和丰富信息。链接永远不会出错，因为 LLM 从不生成它们。

## 实现

```
// 模式：代码收集，LLM 分析

// 步骤 1：确定性收集器（脚本，无 LLM 调用）
collector_run():
  messages = gmail_api.fetch_unread()
  for msg in messages:
    structured = {
      id: msg.id,
      from: msg.sender,
      subject: msg.subject,
      snippet: msg.snippet,
      gmail_link: f"https://mail.google.com/mail/u/?authuser={account}#inbox/{msg.id}",
      gmail_markdown: f"[在 Gmail 中打开]({gmail_link})",
      is_signature: regex_match(msg, DOCUSIGN_PATTERNS),
      is_noise: regex_match(msg, NOISE_PATTERNS),
      is_new: msg.id not in state.seen_ids
    }
    store(structured)
    state.seen_ids.add(msg.id)
  generate_markdown_digest(structured_messages)

// 步骤 2：LLM 读取预格式化的摘要
llm_analyze():
  digest = read("data/digests/today.md")  // 链接已经内嵌
  classify_urgency(digest)                 // 判断调用
  add_commentary(digest)                   // 上下文分析
  run_brain_enrichment(notable_entities)   // gbrain search + 更新
  draft_replies(urgent_items)              // 创意工作
  surface_to_user(final_output)            // 交付

// 步骤 3：连接到 cron
cron_job():
  collector_run()     // 快速、廉价、确定性
  llm_analyze()       // 较慢、昂贵、创造性
```

### 架构

```
+-----------------------------+     +------------------------------+
|      确定性收集器           |---->|        LLM 代理              |
|  (Node.js / Python 脚本)    |     |                              |
|                             |     |  - 读取预格式化的摘要         |
|  - 从 API 拉取数据          |     |  - 分类项目                  |
|  - 存储结构化 JSON          |     |  - 添加评论                  |
|  - 生成链接/URL            |     |  - 运行大脑丰富              |
|  - 检测模式（正则表达式）    |     |  - 草拟回复                  |
|  - 跟踪状态（已见/新）       |     |  - 呈现给用户                 |
|  - 输出 markdown 摘要       |     |                              |
|                             |     |                              |
|  代码 —— 确定性、           |     |  AI —— 判断、上下文、        |
|  永不遗忘                   |     |  创造力                      |
+-----------------------------+     +------------------------------+
```

### 文件结构

```
scripts/email-collector/
├── email-collector.mjs     # 无 LLM 调用，无外部依赖
├── data/
│   ├── state.json          # 上次拉取时间戳、已知 ID、待处理签名
│   ├── messages/           # 每天的结构化 JSON
│   │   └── 2026-04-09.json
│   └── digests/            # 预格式化的 markdown
│       └── 2026-04-09.md
```

### 模式适用场景

| 信号源 | 收集器生成 | LLM 添加 |
|--------|-----------|----------|
| **邮件** | Gmail 链接、发件人元数据、签名检测 | 紧急程度分类、丰富信息、回复草稿 |
| **X/Twitter** | 推文链接、参与度指标、删除检测 | 情感分析、叙事检测、内容创意 |
| **日历** | 事件链接、参会者列表、冲突检测 | 准备简报、从大脑获取会议上下文 |
| **Slack** | 频道链接、线程链接、提及检测 | 优先级分类、行动项提取 |
| **GitHub** | PR/issue 链接、diff 统计、CI 状态 | 代码审查上下文、优先级评估 |

### 原则

如果一段输出必须存在且每次都必须格式正确，用代码生成它。如果一段输出需要判断、上下文或创造力，用 LLM 生成它。不要让 LLM 在同一遍中同时做这两件事。

## 需要注意的地方

1. **LLM 会忘记链接 —— 在代码中内嵌它们。** LLM 会在前 10 个条目遵循"包含 Gmail 链接"规则，然后在第 11 个条目上悄悄漏掉它。再多的提示工程也无法修复长输出上的概率性格式化。解决方案：在收集器脚本中生成每个链接。LLM 读取预格式化的 markdown，其中链接已经嵌入。它不会忘记它没有生成的东西。

2. **噪音过滤必须是确定性的。** 基于正则表达式的噪音检测（新闻通讯、自动收据、营销邮件）属于收集器，而不是 LLM。LLM 可能在一次运行中将新闻通讯分类为"可能重要"，在下一次运行中分类为"噪音"。代码对相同输入的分类每次都相同。

3. **原子写入防止损坏。** 收集器写入跟踪哪些消息已被查看的状态文件（`state.json`）。如果脚本在写入过程中崩溃，状态文件可能损坏。先写入临时文件，然后原子重命名。这也可以防止 cron 在收集运行期间触发时 LLM 读取部分摘要。

## 验证方法

1. **运行收集器并检查每个链接。** 手动执行收集器脚本。打开生成的摘要。点击每个 `[在 Gmail 中打开]` 链接（或等效链接）。每个链接都必须解析到正确的项目。如果任何链接损坏或丢失，收集器有 bug。

2. **验证噪音过滤的一致性。** 对相同输入数据运行收集器两次。噪音分类（is_noise 字段）必须两次都相同。如果不同，概率性元素泄漏到了确定性层。

3. **验证 LLM 读取结构化输出。** 运行完整管道（收集器然后 LLM）。检查 LLM 的分析是否引用了结构化摘要中的数据，而不是它自己生成的数据。最终输出中的链接应该与摘要文件中的链接相同。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*