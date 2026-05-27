# 来源归因

## 目标
大脑中的每个事实都追溯到它的来源——谁说的、在什么背景下、何时说的。

## 用户获得的价值
没有这个：六个月后，有人阅读大脑页面时不知道"Pedro 共同创立了 Brex"是来自 Pedro 本人、LinkedIn 抓取还是幻觉。有了这个：每个声明都可审计，冲突被暴露，大脑是法庭可接受的现实记录。

## 实现方案

```
on brain_write(page, fact):
    # 每个事实都获得引用 — 编译真相和时间线
    citation = format_citation(source)
    #   格式: [Source: {who}, {channel/context}, {date} {time} {tz}]

    # 类别特定格式：
    if source.type == "direct":
        # [Source: User, direct message, 2026-04-07 12:33 PM PT]
    elif source.type == "meeting":
        # [Source: Meeting notes "Team Sync" #12345, 2026-04-03 12:11 PM PT]
    elif source.type == "api_enrichment":
        # [Source: Crustdata LinkedIn enrichment, 2026-04-07 12:35 PM PT]
    elif source.type == "social_media":
        # 必须包含完整 URL — 不仅仅是 @handle
        # [Source: X/@pedroh96 tweet, product launch, 2026-04-07](https://x.com/pedroh96/status/...)
    elif source.type == "email":
        # [Source: email from Sarah Chen re Q2 board deck, 2026-04-05 2:30 PM PT]
    elif source.type == "workspace":
        # [Source: Slack #engineering, Keith re deploy schedule, 2026-04-06 11:45 AM PT]
    elif source.type == "web":
        # [Source: Happenstance research, 2026-04-07 12:35 PM PT]
    elif source.type == "published":
        # [Source: [Wall Street Journal, 2026-04-05](https://wsj.com/...)]
    elif source.type == "funding":
        # [Source: Captain API funding data, 2026-04-07 2:00 PM PT]

    # 将引用内联附加到事实
    gbrain put <slug> --content "...fact [Source: ...]..."

    # 当来源冲突时，记录两者 — 永远不要静默选择一个
    if conflicts_exist(fact, existing_page):
        append_to_compiled_truth(
            "Conflict: Source A says X, Source B says Y. "
            "[Source: A] [Source: B]"
        )

# 冲突解决的来源层次结构（最高权限优先）：
SOURCE_PRIORITY = [
    "User direct statements",      # 1 — 总是获胜
    "Primary sources",             # 2 — 会议、电子邮件、直接对话
    "Enrichment APIs",             # 3 — Crustdata、Happenstance、Captain
    "Web search results",          # 4
    "Social media posts",          # 5
]
```

## 难点

1. **编译真相并非免于引用。** 综合部分中的"Pedro 共同创立了 Brex"与时间线条目一样需要 `[Source: ...]`。大多数代理跳过引用。
2. **推文 URL 是强制性的。** 没有 URL 的 `[Source: X/@handle tweet, topic, date]` 是损坏的引用。当省略 URL 时，数百个大脑页面最终会出现无法访问的推文引用。始终使用：`[Source: X/@handle tweet, topic, date](https://x.com/handle/status/ID)`。
3. **"用户说的"不够。** 在哪里、关于什么、何时。`[Source: User, direct message, 2026-04-07 12:33 PM PT]` — 不仅仅是 `[Source: User]`。
4. **不要静默解决冲突。** 当用户说一件事而 API 说另一件事时，在编译真相中用两个引用记录矛盾。让读者决定。
5. **时间线条目也需要来源。** 时间线的每个追加都带有出处。没有来源的时间线条目是孤儿事实。

## 验证方法

1. 使用 `gbrain get <slug>` 打开任何大脑页面。阅读分隔线上方的编译真相部分。每个事实声明应该有一个内联 `[Source: ...]` 引用。
2. 搜索推文引用：`gbrain search "X/@"`。每个结果应该有完整的 URL，而不仅仅是 @handle。
3. 找到一个来自多个来源的数据的页面（例如，通过 API 丰富的人 + 在会议中提到）。确认两个来源都被独立引用。
4. 检查 3 个随机页面上的时间线条目。每个条目应该有带有日期和上下文的来源引用。
5. 查找用户陈述与 API 结果相矛盾的页面。确认矛盾被记录，而不是静默解决。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*