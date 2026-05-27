# 执行助理模式

## 目标
通过大脑上下文驱动的邮件分类、会议准备和日程安排 —— 使每次互动都能获得完整的关系历史信息。

## 用户获得的价值

没有这个：代理机械地分类邮件（"你有12封未读"），用通用的LinkedIn简介准备会议，并且在没有关系上下文的情况下安排日程。

有了这个：代理在读取邮件正文之前就知道每个发件人是谁，在每次会议前显示共享历史，并根据关系温度和开放线程提供日程建议。

## 实现

```
# 工作流1：邮件分类
on email_batch(emails):
    for email in emails:
        # 步骤1：在读取邮件正文之前搜索发件人
        #   大脑上下文使分类提升10倍
        sender_page = gbrain search "{email.sender_name}"
        if sender_page:
            context = gbrain get <sender_slug>
            #   现在你知道：他们是谁、关系历史、
            #   他们关心什么、开放线程

        # 步骤2：在加载大脑上下文的情况下读取邮件
        #   分类现在是知情的，而不是机械的

        # 步骤3：根据上下文分类
        if context.relationship == "inner_circle" or context.has_open_threads:
            priority = "urgent"
        elif context.is_known_entity:
            priority = "normal"
        else:
            priority = "noise"  # 未知发件人，没有大脑页面

        # 步骤4：根据关系上下文草拟回复
        if needs_reply(email):
            draft = compose_reply(
                email,
                context=context,           # 他们的大脑页面
                open_threads=context.open_threads,  # 你们正在一起做什么
                relationship=context.relationship   # 语气校准
            )

# 工作流2：会议准备
on upcoming_meeting(meeting):
    briefing = {}
    for attendee in meeting.attendees:
        # 为每个参会者搜索大脑
        results = gbrain search "{attendee.name}"
        if results:
            page = gbrain get <attendee_slug>
            briefing[attendee] = {
                "compiled_truth": page.compiled_truth,
                "last_interaction": page.timeline[0],     # 最近的
                "open_threads": page.open_threads,
                "relationship_temperature": page.relationship,
                "relevant_deals": gbrain get_links <attendee_slug>,
            }
        else:
            briefing[attendee] = "没有大脑页面 —— 考虑丰富化"

    # 显示：共享历史、要跟进的内容、要注意的内容
    # "上次你们讨论了B轮时间表。Pedro担心烧钱率。这是他公司页面的最新信息。"

# 工作流3：收件箱后大脑更新
on inbox_cleared():
    for email in processed_emails:
        if email.contained_new_information:
            # 用新信号更新发件人的大脑页面
            gbrain add_timeline_entry <sender_slug> \
                --entry "邮件主题: {subject}。关键信息: {extracted_signal}" \
                --source "来自 {sender} 的邮件关于 {subject}, {date}"

            # 也更新任何提到的实体页面
            for entity in email.mentioned_entities:
                gbrain add_timeline_entry <entity_slug> \
                    --entry "{关于他们说了什么}" \
                    --source "来自 {sender} 的邮件, {date}"

# 工作流4：日程建议
on schedule_request(meeting):
    for attendee in meeting.attendees:
        page = gbrain get <attendee_slug>
        if page.last_interaction > 6_weeks_ago:
            nudge("你已经{weeks}周没见过{attendee}了")
        if page.has_open_threads:
            nudge("{attendee}有一个关于{topic}的开放线程")
        if page.relationship_temperature == "cooling":
            nudge("与{attendee}的关系可能需要关注")
```

## 需要注意的地方

1. **在读取邮件之前搜索发件人。** 这违反直觉但至关重要。首先加载大脑上下文意味着你在看到主题行之前就知道他们是谁、你们正在一起做什么以及他们关心什么。分类是知情的，而不是机械的。
2. **没有大脑页面的未知发件人几乎总是噪音。** 如果`gbrain search`对发件人返回空，他们可能不重要。除非邮件内容另有指示，否则分类为低优先级。
3. **会议准备是最高杠杆的EA工作流。** 用户走进每次会议时已经了解每个与会者的情况：上次互动、开放线程、关系历史。这是"你3点有会议"和"你3点与Pedro有会议——上次你们讨论了B轮融资，他担心烧钱率"之间的区别。
4. **收件箱后大脑更新是大脑复合增长的地方。** 每封邮件都是信号。如果你清空收件箱而不更新大脑页面，信息就会丢失。这是大多数代理跳过的步骤。
5. **日程建议需要时间线数据。** "你已经6周没见过Diana了"只有在会议页面已通过适当的实体传播摄入时才有效（见meeting-ingestion指南）。

## 验证方法

1. 为明天的日历运行会议准备。对于每个参会者，确认代理在生成简报之前运行了`gbrain search`并加载了他们的大脑页面。
2. 分类5封邮件。确认代理在分类邮件之前搜索了大脑中的每个发件人。
3. 清空收件箱后，用`gbrain get <slug>`检查2个发件人大脑页面。确认添加了带有邮件信息的新时间线条目。
4. 检查日程建议。确认代理在建议中引用了参会者的大脑页面（上次互动日期、开放线程）。
5. 从有大脑页面的人发送测试邮件。确认分类响应引用了他们的关系上下文，而不仅仅是邮件内容。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*