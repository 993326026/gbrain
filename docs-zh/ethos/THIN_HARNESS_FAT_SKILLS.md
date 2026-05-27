---
type: essay
title: "Thin Harness, Fat Skills"
subtitle: "How to Make AI Agents Actually Understand Your Data"
author: Garry Tan
created: 2026-04-09
updated: 2026-04-11
tags: [ai, agents, gstack, harness-engineering, skills, architecture]
status: draft-v4
talk: "YC Spring 2026 -- Thin Harness, Fat Skills"
thread: https://x.com/garrytan/status/2042925773300908103
---

# Thin Harness, Fat Skills

Steve Yegge说，使用AI编码代理的人"比使用Cursor和聊天工具的工程师生产力高10倍到100倍，比2005年的Google员工生产力高大约1000倍。"

这是一个真实的数字。我见过，也经历过。但当人们听到100倍时，他们会想：更好的模型。更聪明的Claude。更多参数。

这完全是错误的框架。2倍生产力的人和100倍生产力的人使用的是相同的模型。区别在于五个可以写在索引卡上的概念。

## The harness is the secret sauce

2026年3月31日，Anthropic不小心将Claude Code的全部源代码发布到了npm注册表。512,000行代码。当我读到它时，它证实了我在YC教的一切。秘诀不在于模型本身。而在于包裹模型的东西：the harness。实时仓库上下文。提示缓存。专用工具。上下文膨胀最小化。结构化会话记忆。并行子代理。

这些都不是让模型更聪明。所有这些都是为了在正确的时间给模型提供正确的上下文，而不被噪音淹没。

这才是唯一重要的问题。答案有一个特定的形状。我称之为**thin harness, fat skills**。

## Five definitions

瓶颈永远不是模型的智能。瓶颈在于模型是否理解你的数据模式。模型已经知道如何推理、合成和编写代码。它们失败是因为它们不了解你的数据。五个定义可以解决这个问题。

### Definition 1: Skill File

技能文件是一个可重用的markdown程序，它教模型**如何**做某事。不是**做什么**。用户提供具体内容。技能提供流程。

**Markdown实际上是代码。** 技能文件比僵化的源代码更完美地封装了能力，因为它用模型已经理解的语言描述了流程、判断和上下文。

左边是一个名为`/investigate`的技能。七个步骤：划定数据集范围、构建时间线、整理每份文档、综合分析、论证双方观点、引用来源。它接受三个参数：TARGET、QUESTION和DATASET。

右边是同一技能的两个完全不同的调用。一个指向Sarah Chen博士和210万份发现邮件，询问安全科学家是否被噤声。另一个指向Pacific Corporate Services和FEC文件，询问空壳公司是否在协调竞选捐款。

同一个技能。相同的七个步骤。同一个markdown文件。在一种情况下，它是医学研究分析师。在另一种情况下，它是法医调查员。技能描述了一个判断过程。调用提供了具体场景。

**这是大多数人错过的关键洞察：技能文件像方法调用一样工作。** 它接受参数。你用不同的参数调用它。相同的程序根据你传入的内容产生截然不同的能力。这不是提示工程。这是软件设计，使用markdown作为编程语言，人类判断作为运行时。

### Definition 2: Harness

Harness是运行LLM的程序。它做四件事：在循环中运行模型、读写文件、管理上下文、执行安全检查。这就是"thin"的含义。

反模式是fat harness配合thin skills：40多个工具定义占用了一半的上下文窗口。2到5秒MCP往返的上帝工具。将每个端点都变成工具的REST API包装器。3倍的token消耗，3倍的延迟，3倍的失败率。

相反，你应该构建什么：一个Playwright CLI，每个浏览器操作只需100毫秒。对比：Chrome MCP截图+查找+点击+等待+读取需要15秒。Playwright CLI截图+断言只需200毫秒。快75倍。软件不再需要那么珍贵。只构建你需要的东西。

### Definition 3: Resolver

Resolver是上下文的路由表。当出现任务类型X时，首先加载文档Y。

技能说明**如何**做。Resolver说明**何时**加载**什么**。开发人员修改了一个提示。没有Resolver，他们直接发布。有了Resolver，模型先读取`docs/EVALS.md`，其中写着：运行评估套件，比较分数，如果准确率下降超过2%，回滚并调查。开发人员不知道评估套件的存在。Resolver在正确的时刻加载了正确的上下文。

Claude Code有内置的Resolver。每个技能都有描述字段，模型自动将用户意图与技能描述匹配。你不必记住`/ship`的存在。描述本身就是Resolver。就像Clippy一样。只不过它真的有用。

坦白说：我的CLAUDE.md有20,000行。我遇到的每一件事都写进去了。每一个怪癖，每一个模式，每一个教训。完全荒谬。模型的注意力下降了。Claude Code实际上告诉我要删减它。解决方案：大约200行。只是指向文档的指针。Resolver在重要时加载正确的文档。

### Definition 4: Latent vs. Deterministic

系统中的每一步都是其中之一。

**Latent space**是智能所在的地方。模型读取、解释、决策。判断。合成。模式识别。

**Deterministic**是信任所在的地方。相同输入，相同输出。每次都是。SQL。代码。数字。

LLM可以为8个人安排餐桌座位。让它为800人安排，它会凭空想象出一个看起来合理但完全错误的座位表。这是一个被强行放入潜在空间的确定性问题。最差的系统把错误的工作放在错误的一边。

### Definition 5: Diarization

模型读取关于某个主题的所有内容，并编写结构化的简介。阅读50份文档，产生1页的判断。

没有SQL查询能产生这个。没有RAG管道能产生这个。模型必须实际阅读，记住矛盾，注意变化的内容和时间，并编写结构化的情报。这就是AI对真正的知识工作有用的原因。

## The architecture

三层架构：

**Fat skills**在顶层。Markdown程序编码判断、流程和领域知识。90%的价值在这里。

**Thin CLI harness**在中间层。大约200行。JSON输入，文本输出。默认只读。先CLI，后添加MCP。

**Your app**在底层。QueryDB。ReadDoc。Search。Timeline。确定性基础。

将智能**向上**推入技能。将执行**向下**推入确定性工具。保持harness**thin**。

## The system that learns: YC Startup School

让我向你展示这五个定义如何一起工作。不是理论。而是在YC实际构建的系统中。

Chase Center。2026年7月。6,000名创始人。每个人都有结构化的申请、问卷答案、一对一顾问聊天的记录，以及公共信号：X帖子、GitHub提交、Claude Code记录显示他们的交付速度。

传统方法：15人的项目团队阅读申请，凭直觉做出决定，更新电子表格。在200名创始人时有效。在6,000名时崩溃。

没有人能在工作记忆中记住6,000个资料，并注意到AI代理基础设施队列中三个最佳候选人是拉各斯的开发工具创始人、新加坡的合规创始人，以及布鲁克林的CLI工具创始人，他们在一对一聊天中用不同的语言描述了相同的痛点。

模型可以。

**Step 1: Enrich every founder.**

`/enrich-founder`技能：提取所有来源，运行丰富化，整理成日记格式，突出他们**说**的与他们**实际构建**的内容。在右边，确定性调用：SQL查找过时的资料、GitHub统计数据、演示URL的浏览器测试、社交信号提取、CrustData获取公司情报。

Cron每晚凌晨2点运行。6,000个资料，每晚，始终保持最新。

日记化输出捕捉到任何关键词搜索都找不到的东西：

```
FOUNDER: Maria Santos
COMPANY: Contrail (contrail.dev)
SAYS: "Datadog for AI agents"
ACTUALLY BUILDING: 80% of commits are in billing module.
  She's building a FinOps tool disguised as observability.
```

"说"与"实际构建"。这需要读取GitHub提交历史、申请和顾问记录，并同时记住这三者。

**Step 2: Match 6,000 founders. Make judgment calls.**

这就是技能作为方法调用真正闪耀的地方。三个调用：

`/match-breakout`: 1,200名创始人，按行业亲和力聚类，每房间30人。嵌入+确定性分配。

`/match-lunch`: 600名创始人，机缘匹配（跨行业），每桌8人，不重复。LLM发明主题，然后分配。

`/match-live`: 当前在区域内的任何人，最近邻嵌入，实时200ms，一对一配对，不重复会面。

同一个技能。三个调用。三种完全不同的匹配策略。不同的参数，不同的策略，不同的群体规模。技能描述过程。参数塑造输出。

模型的判断："Santos和Oram都是AI基础设施，但他们不是竞争对手。Santos是成本归因，Oram是编排。把他们放在同一组。"以及："Kim申请的是'开发工具'，但他的一对一记录显示他正在为SOC2构建合规自动化。将他移到FinTech/RegTech。"

没有嵌入能捕捉Kim的重新分类。没有算法能做到这一点。模型必须读取整个资料。

**Step 3: The self-learning loop.**

活动结束后，`/improve`技能读取NPS调查，整理"OK"响应（不是坏的，是平庸的），并提取模式。然后它提出新规则并将它们写回到匹配技能中：

```
When attendee says "AI infrastructure"
    but startup is 80%+ billing code:
    -> Classify as FinTech, not AI Infra.

When two attendees in same group
    already know each other:
    -> Penalize proximity.
       Prioritize novel introductions.
```

这些规则被写回到技能文件中。下次运行时自动使用它们。技能自我重写。

7月活动：12%的"OK"评级。下次活动：4%。技能文件学会了"OK"实际上意味着什么。

与其他领域相同的模式：检索、阅读、日记化、计数、合成。然后：调查、调查、日记化、重写技能。它随处适用。

## OpenClaw: where the skills live

我想再介绍一个harness。不是用于编码的。而是用于其他一切的。

我在OpenClaw上运行一个个人AI代理。它有一个角色，知道我是谁，并维护一个包含数千个相互关联文件的知识库。但让它工作的是完全相同的原则。Thin harness, fat skills。

几天前我在推特上发了这个：

> *不允许做一次性工作。如果我让你做某件事，而这件事以后还需要做，你必须：第一次在3到10个项目上手动完成。给我看输出。如果我批准，将其编码到技能文件中。如果应该自动运行，将其放在cron上。*

> *测试：如果我不得不问你两次，你就失败了。*

这引起了共鸣：一千个赞，两千五百个书签。人们认为这是一个提示工程技巧。不是。这是相同的架构。

Claude Code是最好的编码harness。OpenClaw是最好的其他一切的harness：电子邮件、日历、会议、人员、研究、警报。相同的原则。Thin harness。Fat skills。技能是方法调用。参数改变。过程保持不变。

我写的每个技能都是永久升级。它永远不会退化。它永远不会忘记。它在凌晨3点我睡觉时运行。当下一个模型发布时，每个技能都会立即变得更好，因为潜在步骤中的判断会改善，而确定性步骤保持完美可靠。

这就是你获得Yegge所说的10倍到100倍的方法。不是更聪明的模型。Fat skills，thin harness，以及将一切编码的纪律。

系统会复合增长。构建一次。永远运行。

五个定义。三层架构。一个原则。Thin harness, fat skills.

---

## Agent Decision Guide: Skill or Code?

构建GBrain功能时，请使用此决策指南：

| Question | If YES | If NO |
|----------|--------|-------|
| Does the agent need to think, adapt, or ask questions? | **Skill** (recipe markdown) | Code |
| Same input always produces same output? | **Code** (CLI command) | Skill |
| Does it require judgment about the user's environment? | **Skill** | Code |
| Is it a lookup, list, or status check? | **Code** | Probably skill |
| Does it change behavior based on conversation context? | **Skill** | Code |

**GBrain examples:**
- `gbrain integrations list` = **Code** (reads files, checks env vars, deterministic)
- `gbrain integrations status` = **Code** (checks env vars + heartbeat, deterministic)
- `gbrain integrations doctor` = **Code** (runs health checks, deterministic)
- `gbrain integrations stats` = **Code** (aggregates JSONL, deterministic)
- Recipe setup flow = **Skill** (asks for API keys, adapts to environment, validates)
- Recipe changelog surfacing = **Skill** (agent describes changes conversationally)
- Entity detection = **Skill** (reads message, decides what's important, creates pages)
- Meeting ingestion = **Skill** (reads transcript, extracts entities, updates pages)

**The rule:** If it's a lookup table, it's code. If the agent needs to think, it's a skill.