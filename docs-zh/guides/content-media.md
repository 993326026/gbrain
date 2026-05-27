# 内容和媒体摄取

## 目标
YouTube 视频、社交媒体、PDF 和文档变成可搜索的大脑页面，包含代理自己的分析以及对每个提到的实体的完整交叉引用。

## 用户获得的价值

没有这个：媒体链接只是会失效的书签 —— 你记得看过一个视频，但找不到说了什么、谁说的、为什么重要。

有了这个：每一段媒体都是一个永久的大脑页面，上面叠加了代理的分析，每个提到的实体都有反向链接，全部内容永远可搜索。

## 实现

```
on user_shares_media(url_or_file):

    # 模式 1：YouTube 视频摄取
    if media.type == "youtube":
        # 步骤 1：获取完整的说话人分割转录
        #   谁说了什么 —— 不只是一堵文字墙
        #   使用 Diarize.io 或等效服务
        transcript = diarize(video_url)  # 带说话人属性的转录
        # 绝不使用 YouTube 的自动生成摘要或 AI 摘要

        # 步骤 2：代理编写自己的分析（这是价值所在）
        #   不是摘要。不是复述。是代理的观点：
        #   - 什么重要以及为什么（根据用户的世界观）
        #   - 特定说话人的关键引用
        #   - 与现有大脑页面的连接
        #   - 含义和后续角度
        analysis = agent_analyze(transcript, user_context)

        # 步骤 3：创建大脑页面
        slug = f"media/youtube/{video_slug}"
        gbrain put <slug> --content """
            # {title}
            **频道:** {channel} | **日期:** {date} | **链接:** {url}

            ## 分析
            {agent_analysis}

            ## 关键引用
            - **{Speaker}** ({timestamp}): "{quote}" -- {why_it_matters}

            ---
            ## 完整转录
            {diarized_transcript}
        """

        # 步骤 4：提取并交叉引用实体
        for person in transcript.mentioned_people:
            gbrain add_link <slug> <person_slug>
            gbrain add_link <person_slug> <slug>
            gbrain add_timeline_entry <person_slug> \
                --entry "在 {video_title} 中讨论：{what_was_said}" \
                --source "YouTube: {url}"

    # 模式 2：社交媒体捆绑
    elif media.type == "tweet" or media.type == "social":
        # 不要只保存一条推文 —— 重建完整上下文
        bundle = {
            "original": fetch_tweet(url),
            "thread": reconstruct_thread(url),        # 引用的推文、回复
            "linked_articles": fetch_linked_urls(),    # 获取并总结
            "engagement": get_engagement_data(),       # 什么引起共鸣
        }

        slug = f"media/social/{platform}-{author}-{date}"
        gbrain put <slug> --content """
            # {author}: {topic}
            {agent_analysis_of_full_bundle}

            ## 推文线程
            {reconstructed_thread}

            ## 链接文章
            {article_summaries}

            ---
            ## 原始内容
            {original_tweet_text}
        """

        # 提取实体并交叉引用
        for entity in bundle.mentioned_entities:
            gbrain add_link <slug> <entity_slug>
            gbrain add_link <entity_slug> <slug>

    # 模式 3：PDF 和文档
    elif media.type == "pdf" or media.type == "document":
        # 如果需要则进行 OCR（扫描的 PDF）
        content = ocr_if_needed(file) or extract_text(file)

        # 对于书籍和长篇内容：
        slug = f"sources/{document_slug}"
        gbrain put <slug> --content """
            # {title}
            **作者:** {author} | **日期:** {date}

            ## 章节摘要
            {per_chapter_summary}

            ## 关键引用
            - p.{page}: "{quote}" -- {why_it_matters}

            ## 交叉引用
            {links_to_brain_pages_for_people_and_concepts}

            ---
            ## 来源
            {full_text_or_key_sections}
        """

        for entity in document.mentioned_entities:
            gbrain add_link <slug> <entity_slug>
            gbrain add_link <entity_slug> <slug>

    # 摄取后始终同步
    gbrain sync
```

## 需要注意的地方

1. **始终使用完整转录，绝不使用 AI 摘要。** YouTube 的自动摘要和 AI 生成的摘要会丢失细节：谁说了什么、确切措辞、语气、未说的内容。完整的说话人分割转录本是证据基础。代理的分析放在上面。
2. **代理自己的分析才是价值所在，不是复述。** "视频讨论了 AI 安全"毫无价值。"Dario 提出了关于计算扩展的具体主张，与 Ilya 在 NeurIPS 演讲中所说的相矛盾 —— 见 media/youtube/ilya-neurips-2025"才有用。分析将新媒体与现有大脑连接起来。
3. **社交媒体是一个捆绑包，不是单条推文。** 没有线程、引用推文、链接文章和参与上下文的推文只是一个片段。在创建大脑页面之前重建完整上下文。
4. **交叉引用让媒体页面活起来。** 没有指向提到的人员和公司的反向链接的 YouTube 页面是一个死档案。每个提到的实体都获得一个链接和一个时间线条目。
5. **随着时间推移，`media/` 成为一个可搜索的档案。** 用户消费的每个视频、播客、演讲、采访、文章和推文，上面都有代理的评论。这是完整功能的 memex。

## 验证方法

1. 摄取一个 YouTube 视频。运行 `gbrain get media/youtube/{slug}`。确认页面包含：代理的分析（不仅仅是摘要）、带说话人归属的关键引用、以及完整的说话人分割转录。
2. 运行 `gbrain get_links media/youtube/{slug}`。确认存在指向视频中提到的每个人和公司的大脑页面的反向链接。
3. 选择视频中提到的一个人。运行 `gbrain get <person_slug>`。确认他们的时间线有一个新条目引用该视频并包含具体上下文。
4. 摄取一条推文。确认大脑页面包含线程上下文、链接文章摘要和实体交叉引用 —— 不仅仅是推文文本。
5. 运行 `gbrain search "{topic_from_video}"`。确认媒体页面出现在搜索结果中（验证内容已被索引并可搜索）。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*