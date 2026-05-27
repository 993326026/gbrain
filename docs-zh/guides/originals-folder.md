# Originals 文件夹

## 目标
以用户确切的措辞、深度交叉链接和完整来源记录捕获用户的原创思考——让智力资本积累而不是蒸发。

## 用户获得的价值
没有这个功能：用户在对话中产生了一个出色的框架，但会话结束后就消失了。六个月后，他们隐约记得这个想法，但找不到它，记不起确切的措辞，也无法追溯是什么影响了它。有了这个功能：每条原创观察、论点、框架和热点观点都以原文形式捕获在 `brain/originals/` 中，与塑造它的人物、公司和媒体交叉链接，并永远可搜索。

## 实现方案

```
on user_message(message):
    # 在每条消息中检测原创思考
    if contains_original_thinking(message):
        # 原创性测试：
        #   用户生成的想法？                    -> originals/{slug}.md
        #   用户对他人想法的独特综合？           -> originals/ (综合本身就是原创)
        #   他人创造的世界概念？                 -> concepts/{slug}.md
        #   产品或商业想法？                    -> ideas/{slug}.md

        # 步骤1：使用用户的确切措辞作为slug
        #   "meatsuit-maintenance-tax"
        #   不是 "biological-needs-maintenance-overhead"
        #   生动性就是概念本身。
        slug = slugify(user_exact_phrase)

        # 步骤2：创建originals页面
        gbrain put originals/{slug} --content """
            # {User's Exact Phrase}

            ## The Idea
            {用户的原创思考，用他们自己的话捕获。
             不要意译。不要清理语言。
             原始措辞就是智力产物。}

            ## Context
            {是什么触发了这个思考。会议？文章？对话？
             包括引发它的来源。}
            [来源: 用户, {context}, {date} {time} {tz}]

            ## Connections
            - 相关于: [[{person_slug}]] -- {如何关联}
            - 源于: [[{meeting_slug}]] -- {讨论了什么}
            - 受影响于: [[{book_or_media_slug}]] -- {什么产生了共鸣}
            - 基于: [[{other_original_slug}]] -- {想法如何聚集}
        """

        # 步骤3：交叉链接到所有塑造该思考的事物
        for entity in idea.influences:
            gbrain add_link originals/{slug} <entity_slug>
            gbrain add_link <entity_slug> originals/{slug}

        # 步骤4：同步
        gbrain sync

# 什么算作原创思考：
#   - 新颖框架（"meatsuit maintenance tax"）
#   - 对他人作品的热点观点（综合本身就是原创）
#   - 跨多个实体的模式识别
#   - 对未来的预测或赌注
#   - 有推理的反向观点

# 什么不属于originals/:
#   - 关于世界的事实（-> 实体页面）
#   - 他人创造的概念（-> concepts/）
#   - 产品想法（-> ideas/）
#   - 偏好（-> agent memory）
```

## 难点

1. **命名：生动性就是概念。** 使用 `meatsuit-maintenance-tax` 而不是 `biological-needs-maintenance-overhead`。使用 `ambition-debt` 而不是 `deferred-career-risk-accumulation`。用户生动的措辞就是智力产物。永远不要把它变成企业术语。
2. **综合本身就是原创。** 用户对Peter Thiel的"从0到1"框架的看法应该放在 `originals/`，而不是 `concepts/`。原创部分是用户的综合、解释或异议——即使底层想法来自他人。
3. **没有交叉链接的原创是死的原创。** 连接就是智能。一个关于"抱负债务"的想法如果不链接到体现它的人、讨论它的会议和影响它的书，那就只是坟墓里的一张便条。积极交叉链接。
4. **原创形成集群。** 随着时间推移，用户的想法相互连接。"Meatsuit maintenance tax"连接到"ambition debt"连接到"founder energy budget"。将originals链接到其他originals。集群就是用户的世界观。
5. **捕获触发上下文。** 是什么对话、会议、文章或时刻引发了这个想法？对于未来的检索，上下文往往和想法本身一样重要。将其包含在页面中。

## 验证方法

1. 在对话中产生一个原创想法（例如，"我称之为'抱负债务'问题——你推迟大展宏图的每一年，复利都对你不利"）。确认新页面出现在 `brain/originals/ambition-debt`，使用 `gbrain get originals/ambition-debt`。
2. 检查页面标题和slug是否使用用户的确切措辞——而不是经过清理的版本。
3. 运行 `gbrain get_links originals/ambition-debt`。确认存在到相关人物、会议或其他originals的交叉链接。
4. 表达对他人想法的看法（例如，"我认为Thiel的反向问题是错误的，因为..."）。确认它进入 `originals/`（综合是原创），而不是 `concepts/`。
5. 运行 `gbrain search "ambition debt"`。确认originals页面出现在搜索结果中并且可发现。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*