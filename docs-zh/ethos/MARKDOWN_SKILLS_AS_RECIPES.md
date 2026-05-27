---
type: essay
title: "Homebrew for Personal AI"
subtitle: "Why Markdown is Code and Your Agent is a Package Manager"
author: Garry Tan
created: 2026-04-11
updated: 2026-04-11
tags: [ai, gbrain, gstack, markdown-is-code, open-source, software-distribution, agents, openclaw]
status: draft-v2
prior: "Thin Harness, Fat Skills"
---

# Homebrew for Personal AI

`brew install`给你别人的二进制文件。`npm install`给你别人的源代码。两者都要求你理解工具、配置它、集成它、维护它。

如果软件分发的工作方式不同呢？如果你能用简单的英语描述一种能力，把这个描述交给AI代理，然后代理根据你的设置构建一个原生实现呢？

这就是当markdown是代码时会发生的事情。

## Markdown is code

这是一个真实的技能文件。这个教AI代理筛选电话：

```markdown
# Voice Agent — Your Phone Number

Caller → Twilio → <Stream> WebSocket → Voice Server (port 8765)
                                            ↕ audio
                                      OpenAI Realtime API
                                            ↓ tool calls
                                      Brain / Calendar / Telegram

## Call Routing

Every inbound call routes based on caller phone number + brain lookup:

### Owner → Authenticated Mode
- Send crypto-random 6-digit code to secure channel
- Caller reads it back
- Match → full assistant mode (brain, calendar, scheduling)
- No match → treated as unknown caller

### Known Person, Inner Circle (brain score ≥ 4) → Forward
- Greet by name with brain context
- Transfer to cell
- If no answer (30s timeout), take message
- Text Telegram with who called and context

### Unknown Caller → Screen
- Get their name, look them up in brain
- If inner circle → offer to transfer
- Otherwise → take message
- Create brain entry with phone number (marked UNVERIFIED)
```

那不是伪代码。那不是文档。那是一个工作规范，像Claude Opus 4.6这样拥有百万token上下文窗口的模型可以阅读并实现。架构图告诉它组件。路由表告诉它逻辑。安全模型告诉它约束。代理读取这个文件，理解它，并构建Twilio集成、WebSocket服务器、Telegram机器人钩子、brain查找，所有这些都根据用户已有的基础设施进行调整。

技能文件是一个方法调用。它接受参数（你的电话号码、你的brain、你喜欢的消息应用）。相同的技能，不同的参数，不同的实现。过程就是包。模型就是运行时。

## The distribution mechanism

传统的包管理器分发包：编译的二进制文件、源tarball、容器镜像。消费者运行别人的代码。

GBrain分发配方：markdown文件描述能力，具有足够的特异性，AI代理可以从头实现它们。消费者获得原生实现。没有依赖地狱。没有版本冲突。没有传递性漏洞链。因为没有上游代码。只有关于构建什么和为什么构建的描述。

工作原理如下：

1. **Build a feature.** 实现语音代理、会议摄取管道、电子邮件分类系统、投资尽职调查工作流，无论什么。

2. **GBrain captures the recipe.** 不仅是代码。架构、集成点、失败模式、判断调用。一个编码完整能力的markdown文件。

3. **Push to the repo.** 开源。任何人都可以阅读。

4. **Someone else's agent pulls the recipe.** 读取markdown。说："新配方可用：带来电筛选的AI语音代理。想要吗？"用户说是。代理读取规范并构建它。

无需安装。无需配置向导。无需README。代理阅读文档并弄清楚。

## Why this works now

两年前这不起作用。两件事改变了。

**Context windows hit a million tokens.** 一个用于会议摄取的真实技能文件有200多行。调用它的丰富技能引用brain模式、resolver、引用标准、五个外部API和交叉链接协议。实现这个配方的代理需要在工作记忆中同时保存所有这些内容，同时还要理解用户现有的设置。在8K token时，不可能。在128K时，勉强。在1M时，舒适。

**Models crossed the judgment threshold.** 这是一个来自真实丰富配方的片段：

```markdown
## Philosophy

A brain page should read like an intelligence dossier crossed
with a therapist's notes, not a LinkedIn scrape. We want:

- What they believe — ideology, worldview, first principles
- What they're building — current projects, what's next
- What motivates them — ambition drivers, career arc
- What makes them emotional — angry, excited, defensive, proud
- Their trajectory — ascending, plateauing, pivoting, declining?
- Hard facts — role, company, funding, location, contact info

Facts are table stakes. Texture is the value.
```

实现这个配方的模型必须理解LinkedIn抓取和情报档案之间的区别。这是关于什么信息值得捕获以及如何权衡的判断。GPT-3做不到这一点。GPT-4勉强可以。Opus 4.6做得很好。使能技术是足够聪明的模型，可以解释意图，而不仅仅是遵循指令。

## What a recipe actually contains

一个好的配方有五个部分：

**Architecture.** 组件图。什么与什么对话，通过什么协议，数据流是什么。这是代理首先构建的骨架。

**Routing logic.** 决策树。当X发生时，做Y。当Z失败时，回退到W。这是领域知识所在的地方。语音代理配方编码呼叫路由。尽职调查配方编码如何处理推介材料vs财务模型vs股权表。会议摄取配方编码如何将原始记录转换为可操作的情报。

**Integration points.** 这涉及哪些外部系统？Twilio、Telegram、Gmail、Circleback、Slack、GitHub、Supabase，无论什么。配方命名集成；代理根据用户已配置的内容找出如何连接它们。

**Judgment calls.** 困难的部分。不是"发送电子邮件"而是"根据发件人重要性、时间敏感性以及是否需要决策来决定这封电子邮件是否值得呈现给用户。"跳过判断调用的配方会产生肤浅的实现。判断调用是实际价值所在。

**Failure modes.** 出什么问题以及该怎么办。"如果Circleback令牌过期，向用户发送消息并要求他们重新连接。不要静默跳过。""如果来电显示被欺骗，永远不要信任它进行身份验证。通过单独的渠道使用挑战-响应代码。"没有失败模式的配方会产生脆弱的系统。

这是一个真实的例子。这是尽职调查配方的检测逻辑：

```markdown
## Detection

Recognize data room materials by:
- PDF filenames: "Data Deck", "Intro Deck", "Cap Table",
  "Financial Model", "Pitch Deck", "Series [A-D]"
- Spreadsheets with tabs: Revenue, Retention, Cohorts,
  CAC, Gross Margin, Unit Economics, ARR
- User saying: "data room", "diligence", "deck", "pitch"
- Context: shared in the Diligence topic
```

这是用英语表达的模式匹配器。代理读取这个并知道如何分类传入的文档。没有正则表达式。没有文件类型配置。只是模式的描述和模型关于给定文档是否匹配的判断。

## Pick and choose

GBrain不是单体的。配方是独立的。取你想要的：

- **Voice agent** — 电话筛选、来电显示、brain查找、消息路由
- **Meeting ingestion** — 记录处理、实体提取、行动项捕获、时间线更新
- **Email triage** — 收件箱清理、优先级分类、草稿回复、日程提取
- **Enrichment pipeline** — 从多个数据源进行人员和公司研究，整理到brain页面中
- **Diligence processing** — 数据室摄取、PDF提取、财务模型分析
- **Social monitoring** — X/Twitter时间线分析、提及跟踪、叙事检测
- **Content pipeline** — 想法捕获、链接摄取、文章摘要

每个配方都是自包含的。你的代理知道你已经拥有什么。GBrain每天ping："自上次同步以来有三个新配方。想要任何吗？"你选择。它构建。

而且因为源代码是英语，fork很简单。不喜欢语音代理处理未知来电者的方式？编辑markdown。将"留言"更改为"先问三个筛选问题"。行为改变是因为规范改变了。

## The thin harness, fat skills connection

这篇文章是续集。前传是"Thin Harness, Fat Skills"，它认为100倍AI生产力的秘密不是更好的模型而是更好的上下文管理。保持harness薄（运行模型的程序）。让技能变胖（编码判断和过程的markdown程序）。

"Markdown is code"是分发的必然结果。如果技能是胖markdown文件，如果模型足够聪明，可以从markdown实现，那么技能就是可分发的软件。技能文件同时是：

- **Documentation** for humans reading it
- **Specification** for the implementing agent
- **Package** for the distribution system
- **Source code** for the resulting capability

四个工件合并成一个。这就是为什么这与以前的每个包管理器都不同。`brew install`将配方与二进制文件、文档和源代码分开。GBrain将它们合并。Markdown就是这四个。

## The architecture underneath

三层，与演讲相同：

**Fat skills** on top. Markdown配方编码判断、过程、失败模式和领域知识。这是90%的价值所在。这是被分发的东西。

**Thin harness** in the middle. 运行模型的程序。文件操作、工具调度、上下文管理、安全执行。大约200行。OpenClaw或任何等效的。harness约束越少，配方可以表达的就越多。

**Deterministic foundation** on the bottom. 数据库、API、CLI。相同输入，相同输出，每次都是。SQL查询、HTTP调用、文件读取。技能描述何时调用这些；harness执行它们。

将智能**向上**推入技能。将执行**向下**推入确定性工具。分发技能。这就是整个系统。

## What this means

当实施成本接近零时，瓶颈转移了。不再是"我们能构建这个吗？"而是"我们应该构建这个吗？"和"它到底应该做什么？"

品味、愿景和领域知识成为稀缺资源。深入理解来电筛选并编写精确配方的人比能够从头实现Twilio集成的人创造更多价值。配方就是实现。

这也意味着最好的AI代理设置默认是开源的。封闭的、专有代理配置正在与一个世界竞争，在这个世界里，有人发布一个配方，一千个代理一夜之间实现它。配方以git push的速度传播。护城河是品味，不是代码。

软件分发重新想象：包是markdown文件，运行时是足够聪明的模型，包管理器是你的AI代理，应用商店是git仓库。

`gbrain install voice-agent`

就是这样。