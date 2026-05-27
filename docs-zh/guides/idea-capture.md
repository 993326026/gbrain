# 想法捕获：原创、深度和分布

## 目标

用精确的措辞、深入的上下文和交叉链接捕捉用户的原始思维，使originals文件夹成为大脑中最高价值的内容。

## 用户获得的价值

没有这个：对话中说的精彩想法消失了。代理听到"野心与寿命的比例从未如此破裂"然后忘记了它。

有了这个：每个原始观察都被逐字捕捉，交叉链接到塑造它的人和想法，并根据发布潜力进行评级。你的知识档案随着每次对话而增长。

## 实现

```
capture_idea(message_text, source_context):

  // 1. 归属测试 —— 这个想法属于哪里？
  if user_generated_the_idea(message_text):
    destination = "brain/originals/{slug}.md"
  elif user_synthesis_of_others(message_text):
    destination = "brain/originals/{slug}.md"  // 综合是原创的
  elif world_concept(message_text):
    destination = "brain/concepts/{slug}.md"
  elif product_or_business_idea(message_text):
    destination = "brain/ideas/{slug}.md"
  elif ghostwritten_by_user(message_text):
    destination = "brain/originals/{slug}.md"  // 在元数据中注明代笔
  elif article_about_user(message_text):
    destination = "brain/media/writings/{slug}.md"

  // 2. 用确切措辞捕获 —— 永远不要释义
  page = create_or_update(destination, {
    content: message_text,          // 逐字，不是摘要
    source: source_context,         // 对话、会议、时刻
    reasoning_path: influences,     // 什么导致了洞察力
    depth_context: emotional_nuance // WHAT背后的WHY
  })

  // 3. 原创性评级（对于显著想法）
  if is_notable(message_text):
    rate_originality(page, populations=[
      "general_population", "tech_industry",
      "intellectual_media", "political_establishment"
    ])

  // 4. 交叉链接（强制性 —— 没有链接的原创是死的）
  link_to_people(page, mentioned_people)
  link_to_companies(page, mentioned_companies)
  link_to_meetings(page, source_meeting)
  link_to_media(page, influences)
  link_to_other_originals(page, related_ideas)
  link_to_concepts(page, referenced_concepts)

  // 5. 同步
  gbrain sync --no-pull --no-embed
```

### 归属测试

| 信号 | 目标 |
|------|------|
| 用户生成的想法 | `brain/originals/{slug}.md` |
| 用户对他人想法的独特综合 | `brain/originals/`（综合是原创的） |
| 别人创造的世界概念 | `brain/concepts/{slug}.md` |
| 产品或商业想法 | `brain/ideas/{slug}.md` |
| 用户的代笔书/文章 | `brain/originals/`（在元数据中注明代笔） |
| 关于用户的文章 | `brain/media/writings/` |

### 捕获标准

**使用用户的确切措辞。** 语言就是洞察力。

"The ambition-to-lifespan ratio has never been more broken" 捕捉到了 "tension between ambition and mortality" 没有捕捉到的东西。不要清理它。不要释义。生动的版本才是真实的版本。

**什么值得捕获：**
- 关于世界如何运作的原创观察
- 不同事物之间的新颖联系
- 框架和心智模型
- 模式识别时刻（"我在每个Y中都看到X"）
- 有推理支撑的热点观点
- 揭示新角度的隐喻
- 关于自我或他人的情感/心理洞察

**什么不值得捕获：**
- 日常操作消息（"ok"、"do it"）
- 没有嵌入观察的纯粹问题
- 重复代理说过的话
- 感谢和反应

### 深度测试

**不熟悉用户的人能否阅读此页面并理解他们不仅思考了什么，还有为什么以及如何得出这个结论？**

如果答案是否定的，它需要更多深度。包括：
- 推理路径（导致洞察力的原因）
- 影响因素（他们正在阅读/观看/体验的内容）
- 上下文（对话、会议、时刻）
- 情感或心理细微差别

### 原创性分布评级

对于显著想法，在不同人群中按0-100评级原创性：

```markdown
## 原创性分布

- **普通人群：** 72/100 —— 大多数人没有接触过这个框架
- **科技行业：** 45/100 —— 在创业圈很常见，但对大多数人来说是新颖的
- **知识/媒体阶层：** 68/100 —— 会引起共鸣，尚未被清晰表达
- **政治机构：** 82/100 —— 对政策思维来说完全陌生

**发布信号：** 优秀的文章候选。最佳受众：创始人、建设者。
```

这告诉用户哪些想法值得变成文章、演讲或视频，以及哪个受众会觉得它们最新颖。

### 深度交叉链接要求

**没有交叉链接的原创是死的原创。** 连接就是智慧。

每个原创必须链接到：
- **塑造思维的人**
- **想法发挥作用的公司**
- **讨论它的会议**
- **影响它的书籍和媒体**
- **它连接的其他原创**（想法形成集群）
- **它建立或挑战的概念**

### 显著性过滤

在创建任何实体页面之前，检查显著性：

**创建页面的情况：**
- 你认识或具体讨论的人
- 你正在评估、合作或投资的公司
- 你提到并带有个人反应的媒体
- 你明确参与过的任何人

**不创建页面的情况：**
- 通用引用或附带示例
- 只提到你一次的低互动账户
- 纯粹的隐喻（"like the Roman Empire..."）
- 没有后续的一次性接触

**决策：** 如果显著且没有页面存在，创建一个带网络搜索丰富化的完整页面。不要创建存根。如果你要做一个页面，就要做好。

## 需要注意的地方

1. **综合是原创的。** 当用户以新方式连接两个现有想法时，这种综合属于`brain/originals/`，而不是`brain/concepts/`。新颖的组合是洞察力，即使组成想法不是新的。

2. **确切措辞是不可协商的。** 永远不要释义、总结或"清理"用户的语言。"The ambition-to-lifespan ratio has never been more broken"是洞察力。"Tension between ambition and mortality"是一具尸体。捕获第一个版本。

3. **交叉链接是强制性的，不是可选的。** 没有链接到塑造它的人、公司、会议和概念的原创是死的原创。连接就是智慧。在认为它被捕获之前，检查每个原创至少有2个交叉链接。

## 验证方法

1. **生成一个想法并检查页面。** 在对话中说一些原创的话（例如，"如果markdown文件实际上是分布式软件呢？"）。验证`brain/originals/{slug}.md`以你的确切措辞创建，而不是释义。

2. **检查交叉链接是否存在。** 打开新创建的原创页面。它应该至少链接到提到的人或概念。打开这些链接的页面并验证它们反向链接到原创。

3. **验证深度测试通过。** 像陌生人一样阅读捕获的页面。你能理解用户不仅思考了什么，还有为什么吗？如果推理路径和上下文缺失，捕获就是不完整的。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*