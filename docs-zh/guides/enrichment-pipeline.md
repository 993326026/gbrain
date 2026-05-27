# 实体丰富化管道

## 目标
通过分层花费从外部API丰富大脑页面 —— 关键人物使用完整管道，提及人物轻量级处理，原始数据保留以确保可审计性。

## 用户获得的价值

没有这个：大脑页面只是用户手动输入的单薄外壳，API调用浪费在无关紧要的人身上，丰富化数据在代理会话结束后消失。

有了这个：关键人物拥有丰富的多源画像；花费与重要性成比例；原始API响应被保留用于重新处理；交叉引用连接整个图谱。

## 实现

```
on enrich(entity, trigger):
    # trigger: 会议提及、邮件线程、社交互动、用户请求

    # 步骤1：从传入信号识别实体
    entities = extract_entities(signal)
    #   人物名称、公司名称、关联

    # 步骤2：检查大脑状态 —— 更新还是创建路径？
    for entity in entities:
        existing = gbrain search "{entity.name}"
        if existing:
            page = gbrain get <entity_slug>
            path = "UPDATE"
        else:
            path = "CREATE"

    # 步骤3：确定层级 —— 按重要性缩放花费
    tier = classify_tier(entity):
        # Tier 1（10-15次API调用）：关键人物、核心圈子、商业伙伴、
        #         投资组合公司。完整管道，所有数据源。
        # Tier 2（3-5次API调用）：知名人物、偶尔互动。
        #         网络搜索 + 社交 + 大脑交叉引用。
        # Tier 3（1-2次API调用）：次要提及、其他值得跟踪的人。
        #         大脑交叉引用 + 如果知道用户名则进行社交查找。

    # 步骤4：运行外部查询（按优先级顺序，获得足够信号时停止）
    data = {}
    data["brain"] = gbrain search "{entity.name}"          # 始终首先（免费）
    if tier <= 2:
        data["web"] = brave_search("{entity.name}")        # 背景、新闻、演讲
    if tier <= 2:
        data["twitter"] = twitter_lookup(entity.handle)    # 信念、正在构建的东西、网络
    if tier == 1:
        data["linkedin"] = crustdata_enrich(entity.name)   # 职业、人脉
        data["research"] = happenstance_research(entity)   # 职业轨迹、网络存在
        data["funding"] = captain_api(entity.company)      # 融资、估值、团队
        data["meetings"] = circleback_search(entity.name)  # 转录搜索
        data["contacts"] = google_contacts(entity.email)   # 联系数据

    # 步骤5：存储原始数据（可审计、可重新处理）
    gbrain put_raw_data <entity_slug> \
        --data '{"sources": {"crustdata": {"fetched_at": "...", "data": {...}}, ...}}'
    # 重新丰富化时覆盖，不追加

    # 步骤6：写入大脑页面
    if path == "CREATE":
        gbrain put <entity_slug> --content "<compiled_truth_from_all_sources>"
        gbrain add_timeline_entry <entity_slug> --entry "通过丰富化创建页面"
    elif path == "UPDATE":
        # 追加时间线，仅在有实质性新内容时更新编译真相
        gbrain add_timeline_entry <entity_slug> --entry "丰富化：{new_signal}"
        # 标记矛盾 —— 不要静默解决它们

    # 步骤7：交叉引用图谱
    gbrain add_link <person_slug> <company_slug>       # 人物 -> 公司
    gbrain add_link <company_slug> <person_slug>       # 公司 -> 人物
    gbrain add_link <person_slug> <deal_slug>          # 人物 -> 交易
    # 每个实体页面链接到引用它的每个其他实体页面

# 人物页面部分（不是LinkedIn个人资料 —— 而是一个活的画像）：
#   执行摘要、状态、他们相信什么、他们正在构建什么、
#   什么激励他们、评估、轨迹、关系、联系方式、时间线
# 事实是基本要求。质感才是价值所在。

# 提取质感，而不仅仅是事实：
#   表达了观点？        -> 他们相信什么
#   正在构建或发布？     -> 他们正在构建什么
#   表达了情感？        -> 什么让他们心动
#   他们与谁互动？      -> 网络 / 关系
#   重复出现的话题？    -> 热衷的话题
#   承诺了什么？        -> 开放线程
#   能量水平？          -> 轨迹
```

## 需要注意的地方

1. **不要覆盖人工编写的评估。** 如果用户在Assessment部分写了他们对某人的看法，API丰富化永远不会覆盖它。API数据进入State、Contact、Timeline。用户的评估是神圣不可侵犯的。
2. **同一页面每周丰富化不超过一次。** 在再次运行管道之前检查`put_raw_data`时间戳。丰富化很昂贵，数据不会变化那么快。
3. **LinkedIn连接数 < 20意味着找错人了。** Crustdata有时会返回同名的另一个人。如果LinkedIn个人资料的连接数少于20个，几乎可以肯定是错误匹配。丢弃它。
4. **X/Twitter是最被低估的数据源。** 当你有某人的用户名时，他们的推文揭示了信念、正在构建的东西、兴趣爱好、网络（回复模式）和轨迹（发帖频率、语气变化）。对于"What They Believe"和"What Makes Them Tick"，这比LinkedIn更丰富。
5. **交叉引用不是可选的。** 在丰富化一个人之后，更新他们的公司页面。在丰富化一家公司之后，更新创始人页面。没有交叉链接的丰富化页面是图中的死胡同。

## 验证方法

1. 丰富化一个Tier 1人物。运行`gbrain get <slug>`并确认页面有执行摘要、状态、他们相信什么、联系方式和时间线部分，这些部分从多个来源填充。
2. 运行`gbrain get_raw_data <slug>`。确认原始API响应与`sources.{provider}.fetched_at`时间戳一起存储。
3. 运行`gbrain get_links <slug>`。确认存在指向人物公司页面、交易页面和相关实体的交叉引用链接。
4. 检查一个已丰富化且有用户编写评估的页面。确认评估部分被保留，没有被API数据覆盖。
5. 尝试重新丰富化同一个人。确认系统检查`fetched_at`时间戳，如果不到一周则跳过。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*