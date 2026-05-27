# 大脑优先查询协议

## 目标

在调用任何外部API之前检查大脑。大脑几乎总是有一些东西。外部API填补空白，而不是从头开始。

## 用户获得的价值

没有这个：代理为你已经开了12次会议的人调用Brave Search。你得到的是LinkedIn摘要而不是你的关系历史。

有了这个：代理在做任何其他事情之前先提取你编译的真相、最近的时间线条目和共享上下文。外部API只填补空白。

## 实现

```
lookup(name_or_topic):
  // 步骤 1: 关键字搜索（快速，第一天就能工作，无需嵌入）
  results = gbrain search "{name_or_topic}"
  if results.length > 0:
    page = gbrain get {results[0].slug}
    return page  // 完成，大脑有它

  // 步骤 2: 混合搜索（需要嵌入，找到语义匹配）
  results = gbrain query "what do we know about {name_or_topic}"
  if results.length > 0:
    page = gbrain get {results[0].slug}
    return page

  // 步骤 3: 直接 slug（如果你知道或可以猜测 slug）
  page = gbrain get "people/{slugify(name_or_topic)}"
  if page: return page

  // 步骤 4: 外部API（仅回退）
  // 只有在大脑没有任何内容时才到达这里
  return external_search(name_or_topic)
```

**这是强制性的。** 在检查大脑之前调用Brave Search的代理是在浪费金钱并给出更差的答案。

## 为什么优先使用大脑

大脑拥有外部API无法提供的上下文：
- 关系历史（你如何认识他们，你们讨论过什么）
- 你自己的评估（你对他们的看法，而不是他们的LinkedIn简历）
- 会议记录（说了什么，决定了什么）
- 交叉引用（他们认识谁，他们与哪些公司有联系）
- 时间线（最近发生了什么变化，什么趋势）

LinkedIn抓取给你他们的职位。大脑给你："共同创立了Brex，你和他喝过3次咖啡，上次讨论了支付基础设施论点，他对你对AI代理的看法感兴趣。"

## 需要注意的地方

1. **先尝试关键字，然后混合。** 关键字搜索无需嵌入即可工作（第一天）。混合搜索需要嵌入但能找到语义匹配。按顺序尝试两者。

2. **模糊slug匹配。** `gbrain get` 支持模糊匹配。如果确切的slug不存在，它会建议替代方案。用于名称变体（"Pedro" → "pedro-franceschi"）。

3. **不要因为"简单"问题而跳过。** 即使是"Acme Corp的地址是什么？"也应该先检查大脑。大脑可能有它，并且查找不会增加延迟（关键字搜索< 100ms）。

4. **加载编译真相 + 最近时间线。** 编译的真相在30秒内告诉你当前状态。时间线告诉你最近发生了什么变化。两者结合=完整上下文。

## 验证方法

1. 询问大脑中的某人。验证代理首先搜索大脑（检查响应中的工具调用顺序）。
2. 询问不在大脑中的某人。验证代理搜索了大脑，没有找到，然后回退到外部搜索。
3. 问同一个问题两次。第二次应该是即时的（大脑有它）。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。另见：[大脑-代理循环](brain-agent-loop.md)、[搜索模式](search-modes.md)*