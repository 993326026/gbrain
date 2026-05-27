# 大脑-代理循环

## 目标

每一次对话都会让大脑变得更聪明。每一次大脑查询都会让响应更好。这个循环每天都在复合增长。

## 用户获得的价值

没有这个：代理从陈旧的上下文中回答。你周一讨论了一个交易，到周五代理已经忘记了。每次对话都从零开始。

有了这个：六个月后，代理对你的世界的了解超过了你能在工作记忆中容纳的。它永远不会忘记。它永远不会停止索引。

## 循环流程

```
信号到达（消息、会议、邮件、推文、链接）
  │
  ▼
DETECT 实体（人物、公司、概念、原创思考）
  │  → 生成子代理（参见 entity-detection.md）
  │
  ▼
READ: 首先检查大脑（响应之前）
  │  → gbrain search "{entity name}"
  │  → gbrain get {slug}（如果你知道它）
  │  → gbrain query "what do we know about {topic}"
  │
  ▼
RESPOND 使用大脑上下文（每个答案有上下文更好）
  │
  ▼
WRITE: 更新大脑页面（新信息 → 编译真相 + 时间线）
  │  → gbrain put {slug}（更新页面）
  │  → add_timeline_entry（追加到时间线）
  │  → add_link（交叉引用到其他实体）
  │
  ▼
SYNC: gbrain 索引更改
  │  → gbrain sync --no-pull --no-embed
  │
  ▼
(下一个信号到达 —— 代理现在更聪明了)
```

## 实现

### 每条入站消息

```
on_message(text):
  // 1. DETECT（异步，不阻塞）
  spawn_entity_detector(text)

  // 2. READ（撰写响应之前）
  entities = extract_entity_names(text)  // 快速正则/NER
  context = []
  for name in entities:
    results = gbrain_search(name)
    if results:
      page = gbrain_get(results[0].slug)
      context.append(page.compiled_truth)

  // 3. RESPOND（注入大脑上下文）
  response = compose_response(text, context)

  // 4. WRITE（响应后，如果出现新信息）
  if response_contains_new_info(response):
    for entity in mentioned_entities:
      gbrain_add_timeline_entry(entity.slug, {
        date: today,
        summary: "讨论了 {topic}",
        source: "[来源: 用户，对话，{date}]"
      })

  // 5. SYNC
  gbrain_sync()
```

### 两个不变式

1. **每次读取都改善响应。** 如果你在没有先查看某人的大脑页面的情况下回答了关于他们的问题，你的回答就不如本来可以的那样好。大脑几乎总是有一些东西。外部API填补空白，而不是从头开始。

2. **每次写入都改善未来的读取。** 如果会议记录提到了关于一家公司的新信息，但你没有更新公司页面，你就创造了一个以后会困扰你的空白。

## 需要注意的地方

1. **响应之前读取，而不是之后。** 诱惑是先回应然后更新大脑。但是大脑上下文让响应更好。先读取。

2. **不要跳过写入步骤。** "我稍后会更新大脑"意味着永远不会。在对话后立即写入，趁着上下文还新鲜。

3. **每次写入批次后同步。** 没有同步，大脑搜索索引就是陈旧的。下一次查询不会找到你刚刚写入的内容。

4. **外部API是回退，不是主要。** `gbrain search` 在 Brave Search 之前。`gbrain get` 在 Crustdata 之前。大脑有关系历史、你自己的评估、会议记录、交叉引用。没有外部API可以提供这些。

## 验证方法

1. **提到大脑中已知的人。** 问"我们对{name}了解多少？"代理应该搜索大脑并返回编译的真相，而不是幻想或进行网络搜索。

2. **讨论关于已知实体的新信息。** 说"我听说Acme Corp刚刚完成了B轮融资。"对话后，检查：Acme Corp的大脑页面是否有新的时间线条目？

3. **一天后询问同一个人。** 代理应该立即提取大脑上下文，无需你询问。如果它没有引用大脑页面，说明循环没有运行。

4. **检查同步。** 对话后，从CLI运行`gbrain search "{topic}"`。新信息应该可以搜索到。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。另见：[实体检测](entity-detection.md)、[大脑优先查询](brain-first-lookup.md)*