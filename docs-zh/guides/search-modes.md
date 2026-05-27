# 搜索模式

## 目标
知道何时使用哪种搜索命令 — 关键词、混合或直接 — 以便每次查找都快速并返回正确结果。

## 用户获得的价值
没有这个：代理在搜索命令之间摸索，在需要完整页面时返回块，在直接获取就足够时运行昂贵的语义搜索，或者完全错过结果。有了这个：每次查找都使用最佳模式，尊重 token 预算，用户通过最少的调用获得正确信息。

## 实现方案

```
on user_asks_about(topic):
    # 决策树：选择正确的搜索模式

    if know_exact_slug(topic):
        # 模式3：直接获取 — 即时，无搜索开销
        result = gbrain get <slug>
        # 例如，"告诉我关于 Pedro 的信息" -> gbrain get pedro-franceschi
        # 返回完整页面 — 编译真相 + 时间线

    elif topic.is_exact_name or topic.is_keyword:
        # 模式1：关键词搜索 — 快速，不需要嵌入，第一天即可使用
        results = gbrain search "{name_or_keyword}"
        # 例如，"查找任何关于 Series A 的内容" -> gbrain search "Series A"
        # 返回块，不是完整页面

        # 重要：关键词搜索返回块
        # 如果块确认相关性，则加载完整页面：
        if chunk.confirms_relevance:
            full_page = gbrain get <slug_from_chunk>

    elif topic.is_semantic_question:
        # 模式2：混合搜索 — 语义 + 关键词，需要嵌入
        results = gbrain query "{natural language question}"
        # 例如，"我在金融科技公司认识谁？" -> gbrain query "fintech contacts"
        # 通过向量 + 关键词 + RRF 返回排名块

        # 相同规则：先块，然后根据需要获取完整页面
        if chunk.confirms_relevance:
            full_page = gbrain get <slug_from_chunk>

# 快速参考：
# | 模式    | 命令               | 需要嵌入 | 速度   | 最佳用途                        |
# |---------|-------------------|----------|---------|---------------------------------|
# | 关键词 | gbrain search "term" | 否       | 最快    | 已知名称、精确匹配              |
# | 混合   | gbrain query "..."   | 是       | 快      | 语义问题、模糊匹配              |
# | 直接   | gbrain get <slug>    | 否       | 即时    | 知道 slug 时                    |

# 随时间进展：
#   第1天：关键词搜索（无需嵌入即可工作）
#   第一次嵌入后：解锁混合搜索
#   知道 slug 后：直接获取以提高速度

# 页面内冲突信息的优先级：
#   1. 用户的直接陈述（总是获胜）
#   2. 编译真相部分（从证据综合）
#   3. 时间线条目（原始信号，逆时间顺序）
#   4. 外部来源（网络搜索、API）
```

## 难点

1. **搜索返回块，不是完整页面。** 在 `gbrain search` 或 `gbrain query` 之后，你会得到摘录。当块确认相关性时，始终运行 `gbrain get <slug>` 来加载完整页面。当完整上下文很重要时，不要仅从块回答问题。
2. **关键词搜索无需嵌入即可工作。** 在任何嵌入运行之前的第一天，`gbrain search` 仍然可以工作。不要告诉用户"搜索尚未可用"——关键词搜索始终可用。
3. **不要对已知名称使用混合搜索。** `gbrain query "Pedro Franceschi"` 浪费嵌入计算。使用 `gbrain search "Pedro Franceschi"`，如果知道 slug，更好的是使用 `gbrain get pedro-franceschi`。
4. **Token 预算意识。** 通过 `gbrain get` 获取的完整页面可能很大。先读取搜索块以确认相关性，然后再提取完整页面。"有人提到 Series A 吗？"——搜索结果（块）可能足够。"告诉我关于 Pedro 的一切"——获取完整页面。
5. **混合搜索需要已运行嵌入。** 如果 `gbrain query` 返回空但 `gbrain search` 找到结果，说明嵌入尚未生成。先运行嵌入管道。

## 验证方法

1. 运行 `gbrain search "Pedro"` — 确认它返回包含匹配文本和 slug 引用的块。
2. 运行 `gbrain query "who works at fintech companies"` — 确认它返回语义相关结果（不仅仅是"fintech"上的关键词匹配）。
3. 运行 `gbrain get pedro-franceschi` — 确认它返回包含编译真相和时间线的完整页面。
4. 比较：使用所有三种模式搜索同一实体。关键词应该最快，混合应该显示概念匹配，直接应该返回完整页面。
5. 搜索返回块后，对该块的 slug 运行 `gbrain get`。确认完整页面包含比块本身更多的上下文。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*