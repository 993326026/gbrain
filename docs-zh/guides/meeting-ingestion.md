# 会议摄取

## 目标
会议记录变成大脑页面，更新每个提到的实体 —— 参会者、公司、交易和行动项都在一次传递中传播。

## 用户获得的价值

没有这个：会议消失在记忆中，行动项被遗忘，代理不知道上次你与某人见面时讨论了什么。

有了这个：每次会议都是永久记录，丰富它触及的每个人和公司页面，用户走进每次跟进会议时已经了解情况。

## 实现

```
on new_meeting_transcript(meeting):
    # 步骤1：获取完整的记录 —— 不是AI摘要
    #   AI摘要会产生幻觉框架（"同意了..."）
    #   记录是事实真相
    transcript = fetch_full_transcript(meeting.id)  # 例如，Circleback API
    # 必须有说话人分割：谁在说什么

    # 步骤2：创建会议页面
    slug = f"meetings/{meeting.date}-{short_description}"
    compiled_truth = agent_analysis(transcript):
        # 线上：代理自己的分析，不是通用回顾
        #   - 通过用户的优先级重新构建框架
        #   - 标记意外、矛盾、含义
        #   - 命名真正的决策（不是表演性的）
        #   - 指出未说或未解决的问题
    timeline = format_diarized_transcript(transcript)
        # 线下：完整记录，仅追加
        #   格式：**Speaker** (HH:MM:SS): Words.

    gbrain put <slug> --content "<compiled_truth>\n---\n<timeline>"

    # 步骤3：传播到所有实体页面（强制性 —— 大多数代理跳过这一步）
    for person in meeting.attendees + meeting.mentioned_people:
        gbrain add_timeline_entry <person_slug> \
            --entry "在'{meeting.title}'会议上会面于{date}。要点：..." \
            --source "会议记录'{meeting.title}', {date}"
        # 如果出现新信息，更新他们的状态部分
        # 如果相关，更新每个人公司的公司页面

    for company in meeting.mentioned_companies:
        gbrain add_timeline_entry <company_slug> \
            --entry "在'{meeting.title}'中讨论：{讨论内容}" \
            --source "会议记录'{meeting.title}', {date}"

    # 步骤4：提取行动项
    action_items = extract_action_items(transcript)
    # 添加到任务列表并归属责任人

    # 步骤5：反向链接所有内容（双向图）
    for entity in all_entities_mentioned:
        gbrain add_link <slug> <entity_slug>   # 会议 -> 实体
        gbrain add_link <entity_slug> <slug>    # 实体 -> 会议

    # 步骤6：同步以便新页面立即可搜索
    gbrain sync

# 调度：cron每天3次（上午10点、下午4点、晚上9点）以捕获新会议
# 来源：Circleback (https://circleback.ai) 或任何具有
#         说话人分割 + API/webhook访问的服务
```

## 需要注意的地方

1. **始终获取完整记录，永不获取AI摘要。** AI摘要会产生幻觉框架 —— 它们会编辑"同意"或"决定"的内容，即使没有达成这样的协议。说话人分割的记录是事实真相。
2. **实体传播是大多数代理跳过的步骤。** 直到每个参会者的页面、每个提到的人的页面和每个公司的页面都有新的时间线条目，会议才算完全摄取。没有传播的会议页面本身是无用的。
3. **提到的人不仅仅是参会者。** 如果会议讨论了"Sarah在Brex的团队"，那么Sarah的页面和Brex的页面都需要更新 —— 即使Sarah不在房间里。
4. **代理的分析是价值所在，不是摘要。** "他们讨论了Q2目标"毫无价值。"Pedro对烧钱率提出异议，Diana没有承诺时间表，没有人解决定价差距"才有用。
5. **反向链接必须是双向的。** 会议页面链接到参会者页面，参会者页面也链接回会议。图是双向的。永远。

## 验证方法

1. 摄取会议后，运行`gbrain get meetings/{date}-{slug}`。确认页面上方有代理的分析，下方有完整的说话人分割记录。
2. 对于每个参会者，运行`gbrain get <attendee_slug>`。检查他们的时间线是否有引用会议的新条目，并包含具体见解（不仅仅是"参加会议"）。
3. 选择会议中提到的一家公司。运行`gbrain get <company_slug>`。确认存在引用关于公司讨论内容的时间线条目。
4. 运行`gbrain get_links meetings/{date}-{slug}`。验证存在指向所有参会者和实体页面的反向链接。
5. 运行`gbrain search "{meeting_topic}"`。确认会议页面出现在搜索结果中（验证同步已运行）。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*