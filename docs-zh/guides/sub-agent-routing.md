# 子代理模型路由

## 目标

将子代理路由到能够完成工作的最便宜模型，在不牺牲质量的情况下节省 10-40 倍的成本。

## 用户获得的价值

没有这个：每个子代理都在 Opus 上运行（$15/MTok）。每条消息的实体检测成本为 $3-5/天。研究任务每项成本 $10+。

有了这个：实体检测在 Sonnet 上运行（$3/MTok，便宜 5 倍）。研究在 DeepSeek 上运行（$0.50/MTok，便宜 30 倍）。主会话保持在 Opus 上以保证质量。总成本下降 70-80%。

## 实现方案

### 路由表

| 任务类型 | 推荐模型 | 原因 |
|----------|----------|------|
| 主会话 / 复杂指令 | Opus-class（默认） | 最佳推理和指令遵循 |
| 研究 / 综合 / 分析 | DeepSeek V3 或同等 | 便宜 25-40 倍，探索性工作表现出色 |
| 结构化输出 / 长上下文 | 大上下文模型（Qwen, Gemini） | 200K+ 上下文，可靠的 JSON 输出 |
| 快速轻量级子代理 | 快速推理模型（Groq） | 500 tok/s，便宜，适合快速任务 |
| 深度推理（谨慎使用） | 推理模型（DeepSeek-R1, o3） | 最适合难题，昂贵 |
| 实体检测（信号检测器） | Sonnet-class | 快速，便宜，检测质量足够 |

### 信号检测器模式

在每条入站消息上生成轻量级子代理。这是强制性的。

```
on_every_message(text):
  // 异步生成 — 不阻塞响应
  spawn_subagent({
    task: `SIGNAL DETECTION — scan this message:
    "${text}"

    1. IDEAS FIRST: Is the user expressing an original thought?
       If yes -> create/update brain/originals/ with EXACT phrasing
    2. ENTITIES: Extract person names, company names, media titles
       For each -> check brain, create/enrich if notable
    3. FACTS: New info about existing entities -> update timeline
    4. CITATIONS: Every fact needs [Source: ...] attribution
    5. Sync changes to brain repo`,
    model: "sonnet-class",  // fast + cheap
    timeout: 120s
  })
```

**为什么使用 Sonnet-class 进行检测：** 实体检测是模式匹配，而不是深度推理。Sonnet 比 Opus 便宜 5-10 倍，并且对于异步检测足够快。主会话在检测并行运行时继续在 Opus 上运行。

### 研究管道模式

对于研究密集型任务，使用多模型管道：

```
1. PLANNING (Opus):     编写研究摘要，确定要查找的内容
2. EXECUTION (DeepSeek): 子代理执行实际研究（网络、API、文档）
3. SYNTHESIS (Opus):     读取研究输出，添加战略分析
```

**为什么这样有效：** 规划和综合步骤需要品味和判断力（Opus）。执行步骤是机械数据收集（DeepSeek，成本低 25-40 倍）。你以 DeepSeek 级别的成本获得 Opus 质量的输出，用于 80% 的工作。

### 何时生成子代理

| 情况 | 生成？ | 模型 |
|------|--------|------|
| 每条入站消息 | 是（强制） | Sonnet |
| 研究请求 | 是 | DeepSeek 用于执行 |
| 快速查找 / 事实检查 | 是 | 快速模型（Groq） |
| 复杂分析 | 否 — 在主会话中处理 | Opus |
| 写作 / 编辑 | 否 — 在主会话中处理 | Opus |

### 成本优化

主会话在最佳模型上运行。其他所有内容都在能够完成工作的最便宜模型上运行。实际上，60-70% 的子代理工作是实体检测（Sonnet）和研究执行（DeepSeek），比主会话模型便宜 10-40 倍。

## 难点

1. **检测使用 Sonnet，而不是 Opus。** 最常见的错误是在 Opus 上运行实体检测。检测是模式匹配，不是深度推理。Sonnet 便宜 5-10 倍且足够快。将 Opus 保留用于推理质量至关重要的主会话。

2. **不要阻塞主线程。** 子代理必须异步运行。如果信号检测器同步运行，用户在实体检测完成时每条消息等待 30-120 秒。生成并忘记。用户立即看到响应。

3. **成本优化是乘法的。** 实体检测在每条消息上运行。如果你在每天 50 条消息上使用 $15/MTok 的 Opus 进行检测，仅检测就需要 $3-5/天。$3/MTok 的 Sonnet 将其降至 $0.60-1.00/天。一个月内，错误的模型选择比必要的成本高出 $100+。

## 验证方法

1. **生成信号检测器并检查模型。** 发送消息并验证子代理在 Sonnet-class 上生成，而不是 Opus。检查子代理配置或日志中的模型字段。

2. **检查每日成本。** 在子代理路由运行一天后，将总 API 成本与之前没有路由的一天进行比较。你应该看到总成本减少 50-80%。

3. **验证异步执行。** 发送消息并测量响应时间。响应应在 5 秒内到达。如果需要 30+ 秒，信号检测器正在同步运行并阻塞主线程。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*