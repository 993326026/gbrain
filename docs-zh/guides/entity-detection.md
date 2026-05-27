# 实体检测：在每条消息上运行

## 目标

每一条入站消息都会被扫描以查找原始思维和实体提及，因此大脑会在每次对话中自动增长。

## 用户获得的价值

没有这个：代理回答问题但忘记一切。你在会议中提到Pedro，下周代理不知道Pedro是谁。

有了这个：对话中提到的每个人、公司和想法都会得到一个大脑页面。下次Pedro出现时，代理已经有了上下文。大脑不断复合增长。

## 实现

在每一条入站消息上生成一个轻量级子代理。不要等待它完成再响应。它并行运行。

```
on_every_message(message_text, source_context):

  // 1. 异步生成 —— 不要阻塞响应
  spawn_subagent({
    model: "sonnet-class",     // 便宜 + 快速，不是 opus
    timeout: 120,              // 秒
    task: build_detection_prompt(message_text, source_context)
  })

  // 2. 正常响应用户
  // 子代理在后台运行
```

### 检测提示词

```
build_detection_prompt(text, source):
  return `
信号检测 —— 扫描此消息以查找想法和实体：

消息: "${text}"
来源: [来源: 用户, ${source.topic}, ${source.platform}, ${source.timestamp}]

步骤1 —— 想法优先（最高优先级）：
用户是否表达了原始思想、观察、论文或框架？

如果是：
  - 创建或更新 brain/originals/{slug}.md
  - 使用用户的确切措辞（语言就是洞察力）
  - "The ambition-to-lifespan ratio has never been more broken" 比
    "tension between ambition and mortality" 更好
  - 包含 [来源: ...] 引用和完整上下文

如果想法引用了世界概念：brain/concepts/{slug}.md
如果是产品/商业想法：brain/ideas/{slug}.md

步骤2 —— 实体：
提取所有人名、公司名、媒体标题。

对于每个实体：
  a. 运行：gbrain search "{name}"
  b. 如果页面存在且有新信息：追加时间线条目
     格式: - YYYY-MM-DD | {发生了什么} [来源: {谁}, {上下文}, {日期}]
  c. 如果没有页面且实体显著：创建带网络丰富化的页面
  d. 如果页面单薄（编译真相 < 5行）：生成后台丰富化

步骤3 —— 反向链接（强制性）：
对于每个提到的实体，添加从其页面到此来源的反向链接。
未链接的提及是破碎的大脑。
格式: - **YYYY-MM-DD** | 在 [{页面标题}]({路径}) 中引用 — {上下文}

步骤4 —— 同步：
运行: gbrain sync --no-pull --no-embed

如果没有要捕获的内容，回复"No signals detected"并退出。
`
```

### 显著性过滤

在创建新实体页面之前，检查显著性：

```
is_notable(entity):
  // 创建页面的情况：
  - 用户认识或具体讨论的人
  - 用户正在评估、合作或投资的公司
  - 用户提到并带有个人反应的媒体
  - 用户明确参与过的任何人

  // 不创建页面的情况：
  - 通用引用或附带示例
  - 只提到用户一次的低互动账户
  - 纯粹的隐喻（"like the Roman Empire..."）
  - 没有后续的一次性接触

  // 如果显著且没有页面：创建完整页面（不是存根）
  // 如果不显著：静默跳过
```

### 什么算作原创思维

| 捕获 | 不捕获 |
|------|--------|
| 关于世界如何运作的原创观察 | "ok"、"do it"、"sure" |
| 不同事物之间的新颖联系 | 没有观察的纯粹问题 |
| 框架和心智模型 | 重复代理说过的话 |
| 模式识别（"我在每个Y中都看到X"） | 感谢和反应 |
| 有推理的热点观点 | 日常操作消息 |
| 揭示新角度的隐喻 | 没有嵌入洞察力的请求 |

### 归档规则

| 信号 | 目标 |
|------|------|
| 用户生成的想法 | `brain/originals/{slug}.md` |
| 用户对他人想法的综合 | `brain/originals/`（综合是原创的） |
| 别人创造的世界概念 | `brain/concepts/{slug}.md` |
| 产品或商业想法 | `brain/ideas/{slug}.md` |
| 提到的人 | `brain/people/{slug}.md` |
| 提到的公司 | `brain/companies/{slug}.md` |
| 引用的媒体 | `brain/media/{type}/{slug}.md` |

### 反向链接铁律

每个实体提及必须创建从实体页面到来源的反向链接。这不是可选的。

```
// 当消息提到"Pedro"并创建会议页面时：

// 1. 更新会议页面（正常）
brain/meetings/2026-04-10-board-sync.md:
  - Pedro presented Q1 numbers

// 2. 也更新 Pedro 的页面（反向链接）
brain/people/pedro-franceschi.md:
  ## 时间线
  - **2026-04-10** | 在董事会同步会议上展示了Q1数字
    [来源: 用户, 董事会会议, 2026-04-10]
```

没有反向链接，你无法遍历图谱。"显示与Pedro相关的所有内容"只有在Pedro的页面链接回每个提及时才有效。

## 需要注意的地方

1. **不要阻塞对话。** 实体检测异步运行。用户应该立即看到响应，而不是等待2分钟让子代理丰富5个实体页面。

2. **Sonnet，不是Opus。** 实体检测是模式匹配，不是深度推理。Sonnet便宜5-10倍且足够快。主要对话使用Opus。

3. **确切措辞很重要。** "Markdown is actually code"是一个洞察力。"Markdown can be used as code"是一个摘要。捕获第一个版本。

4. **不要创建存根。** 如果你创建页面，就要做好。运行网络搜索，构建编译真相，添加上下文。只有名字的存根页面比没有页面更糟（它给出错误的信心）。

5. **创建前去重。** 创建页面之前总是运行`gbrain search`。变体拼写、昵称和公司缩写会导致重复。"Pedro Franceschi"和"Pedro"可能是同一个人。

## 验证方法

1. **发送一条提到某人的消息。** 说"I had coffee with Sarah Chen from Acme Corp today."验证：brain/people/sarah-chen.md被创建或更新，brain/companies/acme-corp.md被创建或更新，两者都有今天日期的时间线条目。

2. **发送一条包含原创想法的消息。** 说"What if we could distribute software as markdown files that agents execute?"验证：brain/originals/{slug}.md以你的确切措辞创建。

3. **检查反向链接。** 打开Sarah Chen的页面。它应该有一个链接回今天对话的时间线条目。打开Acme Corp的页面。同样。

4. **发送一条无聊的消息。** 说"ok sounds good."验证：没有创建任何内容。检测器应该报告"No signals detected."

5. **检查重复项。** 先提到"Pedro"，然后提到"Pedro Franceschi."验证：一个页面，不是两个。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*