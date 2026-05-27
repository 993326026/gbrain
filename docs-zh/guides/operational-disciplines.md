# 运营规范

## 目标
区分生产环境大脑与演示大脑的五条不可协商规则——信号检测、大脑优先查询、每次写入后同步、每日心跳检查和夜间梦境周期。

## 用户获得的价值
没有这些规范：代理会错过对话中的信号，在大脑已有答案时仍浪费钱调用外部API，写入后搜索结果过时，大脑悄悄"腐烂"。有了这些规范：每条消息都被扫描以检测实体，始终先查询大脑，搜索结果始终最新，健康状态每日监控，大脑在夜间自动进化。

## 实现方案

```
# 规范1：每条消息都进行信号检测（强制）
on every_inbound_message(message):
    # 无一例外。如果用户自言自语但大脑没有捕获，系统就出问题了。这是第1条规范。

    entities = detect_entities(message)
    #   人物、公司、交易、原创想法

    for entity in entities:
        existing = gbrain search "{entity.name}"
        if existing:
            gbrain add_timeline_entry <entity_slug> \
                --entry "{what_was_said}" \
                --source "User, direct message, {timestamp}"
        # else: 如果足够重要，标记为需要丰富化

    originals = detect_original_thinking(message)
    for idea in originals:
        gbrain put originals/{slug} --content "{user's exact phrasing}"

# 规范2：调用外部API前先进行大脑优先查询（强制）
on information_needed(topic):
    # 始终先检查大脑再访问网络
    brain_result = gbrain search "{topic}"
    if brain_result:
        page = gbrain get <slug>
        # 优先使用大脑数据。外部API只是填补空白，而非替代。
    else:
        # 大脑中没有相关内容——现在使用外部API
        external_result = brave_search("{topic}")

    # 在检查自己的大脑之前就去调用网络的代理，既浪费钱又给出更差的答案。

# 规范3：每次写入后同步（强制）
on brain_write_complete():
    gbrain sync
    # 没有同步，搜索结果就会过时。
    # 刚写入的页面在同步运行前不会出现在 gbrain search 或 gbrain query 中。
    # 跳过同步意味着下次查询会错过最新数据。

# 规范4：每日心跳检查
on daily_schedule("09:00"):
    gbrain doctor
    # 检查：数据库连接、嵌入健康状态、同步状态、
    # 页面数量、过时页面、损坏链接
    # 如果doctor报告问题，先修复再做其他事情。

# 规范5：夜间梦境周期
on nightly_schedule("02:00"):
    # 梦境周期是最重要的规范。
    # 大脑在夜间自我进化。

    # 5a: 实体扫描——找到未链接的提及
    pages = gbrain list_pages
    for page in pages:
        mentions = extract_entity_mentions(page.content)
        existing_links = gbrain get_links <page.slug>
        for mention in mentions:
            if mention not in existing_links:
                gbrain add_link <page.slug> <mention_slug>  # 修复破损的知识图谱

    # 5b: 引用审计——找到无来源的事实
    for page in pages:
        facts_without_sources = audit_citations(page.content)
        if facts_without_sources:
            flag_for_remediation(page, facts_without_sources)

    # 5c: 记忆巩固——从时间线更新编译真相
    for page in stale_pages(older_than="7d"):
        timeline = gbrain get_timeline <page.slug>
        if timeline.has_new_entries_since_last_consolidation:
            # 从累积的时间线重新合成编译真相
            updated_truth = consolidate(page.compiled_truth, timeline.new_entries)
            gbrain put <page.slug> --content updated_truth

    # 5d: 同步所有内容
    gbrain sync

# 额外建议：持久技能优于一次性工作
# 如果做某事两次，就把它做成技能 + cron任务。
#   1. 构思流程
#   2. 手动运行3-10个项目
#   3. 修订——迭代优化质量
#   4. 编写成技能
#   5. 添加到cron——自动化
# 每种实体类型和信号源只有一个所有者技能。
# 两个技能创建同一页面 = 覆盖违规。
```

## 难点

1. **梦境周期是最重要的规范。** 大脑在夜间自我进化。实体扫描修复破损的知识图谱，引用审计捕获无来源的事实，记忆巩固保持编译真相最新。跳过梦境周期，大脑会慢慢"腐烂"。
2. **跳过规范3（写入后同步）意味着搜索结果过时。** 你写入一个页面，然后立即搜索它——却什么也找不到。页面存在但未被索引。写入后始终要同步。
3. **信号检测必须在每条消息上触发。** 不仅仅是看起来重要的消息。用户随口说"我昨天和Pedro谈了董事会席位的事"——这应该成为Pedro页面上的时间线条目，可能更新他的状态部分，也是关于董事会的信号。如果代理没捕获到，系统就出问题了。
4. **大脑优先既省钱又给出更好的答案。** 大脑拥有外部API没有的上下文：关系历史、会议记录、用户自己的评估。对"Pedro Franceschi"的API查询返回LinkedIn资料。大脑返回包括私人上下文在内的完整画面。
5. **`gbrain doctor` 捕获静默故障。** 嵌入管道可能停滞，同步可能静默失败，数据库连接可能断开。每日心跳检查在这些问题复合成数据丢失之前捕获它们。

## 验证方法

1. 发送一条提及大脑中有人物页面的消息。确认代理检测到实体并在其页面上添加时间线条目（`gbrain get_timeline <slug>`）。
2. 向代理询问大脑中的某人。确认它在调用外部API之前运行 `gbrain search` 或 `gbrain get`（检查工具调用顺序）。
3. 使用 `gbrain put` 写入新页面，然后立即运行 `gbrain search` 搜索它。确认它出现在结果中（验证同步已运行）。
4. 运行 `gbrain doctor`。确认它返回包含数据库状态、页面数量和任何标记问题的健康报告。
5. 梦境周期运行后，检查有未链接实体提及的页面。确认已添加新链接（`gbrain get_links <slug>`）。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*