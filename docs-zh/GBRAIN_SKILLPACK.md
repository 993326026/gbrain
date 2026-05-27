<!-- skillpack-version: 0.7.0 -->
<!-- source: https://raw.githubusercontent.com/garrytan/gbrain/master/docs/GBRAIN_SKILLPACK.md -->
# GBrain Skillpack: AI 代理参考架构

这是一份参考架构，展示生产级 AI 代理如何使用 gbrain 作为其知识骨干。基于真实部署的模式，该部署拥有 14,700+ 大脑文件、40+ 技能和 20+ 持续运行的 cron 作业。

**Memex 愿景实现。** Vannevar Bush 设想了一种设备，个人可以存储一切，机械化以便可以以极快的速度查阅。GBrain 就是那种设备，除了 memex 自己构建自己。代理检测实体、丰富页面、创建交叉引用并自动维护编译真相。

下面每个部分都是独立的指南。点击查看完整内容。

---

## 核心模式

基础读写循环和数据模型。

| 指南 | 涵盖内容 |
|-------|---------------|
| [大脑-代理循环](guides/brain-agent-loop.md) | 使大脑随时间复合的读写周期 |
| [实体检测](guides/entity-detection.md) | 在每条消息上运行。捕获原始想法 + 实体提及 |
| [原始文件夹](guides/originals-folder.md) | 捕获你思考的内容，而不仅仅是你找到的内容 |
| [大脑优先查找](guides/brain-first-lookup.md) | 在调用任何外部 API 之前检查大脑 |
| [编译真相 + 时间线](guides/compiled-truth.md) | 线上方：当前综合。线下：仅追加证据 |
| [来源归因](guides/source-attribution.md) | 每个事实都需要引用。格式和层次结构 |

## 数据管道

获取数据并保持最新。

| 指南 | 涵盖内容 |
|-------|---------------|
| [丰富管道](guides/enrichment-pipeline.md) | 7 步协议，层级系统（按重要性分为 Tier 1/2/3） |
| [会议摄取](guides/meeting-ingestion.md) | 始终提取完整记录，传播到所有实体页面 |
| [内容与媒体摄取](guides/content-media.md) | YouTube、社交媒体捆绑包、PDF/文档 |
| [尽职调查摄取](guides/diligence-ingestion.md) | 数据室材料：推介演示、财务模型、股权表 |
| [确定性收集器](guides/deterministic-collectors.md) | 代码用于数据，LLM 用于判断。收集器模式 |
| [想法捕获与原创](guides/idea-capture.md) | 深度测试、原创性分布、深度交叉链接 |
| [获取数据](integrations/README.md) | 集成方案：语音、电子邮件、X、日历 |

## 运营

运行生产级大脑。

| 指南 | 涵盖内容 |
|-------|---------------|
| [参考 Cron 计划](guides/cron-schedule.md) | 20+ 定期作业、安静时间、梦想周期 |
| [通过 Minions 执行 Cron](../skills/conventions/cron-via-minions.md) | 为什么计划任务作为 Minion 作业运行，而不是 `agentTurn`。v0.11.0 迁移自动应用于内置处理程序；特定于主机的处理程序使用下面的插件契约。 |
| [插件处理程序](guides/plugin-handlers.md) | 通过代码注册特定于主机的 Minion 处理程序（无数据文件执行表面）。 |
| [Minions 修复](guides/minions-fix.md) | 修复半迁移的 v0.11.0 安装。 |
| [Shell 作业（v0.14.0+）](guides/minions-shell-jobs.md) | 将确定性 cron（API 获取、令牌刷新、抓取+写入）移出 LLM 网关。每次触发零令牌，网关容量增加约 60%。遵循 `skills/migrations/v0.14.0.md` 了解采用指南。 |
| [安静时间与时区](guides/quiet-hours.md) | 在睡眠期间保持通知，时区感知交付 |
| [执行助理模式](guides/executive-assistant.md) | 电子邮件分类、会议准备、日程安排 |
| [运营规范](guides/operational-disciplines.md) | 信号检测、大脑优先、写后同步、心跳、梦想周期 |
| [技能开发周期](guides/skill-development.md) | 5 步周期：概念、原型、评估、编码、cron |

**子代理路由（v0.11.0+）：** 调度后台工作的代理应通过 `skills/conventions/subagent-routing.md` 路由 —— 它读取 `~/.gbrain/preferences.json#minion_mode` 并在原生子代理和 Minion 作业之间分支。v0.11.0 迁移自动向 AGENTS.md 注入指向此约定的标记。

**Cron 路由（v0.11.0+）：** 计划任务通过 Minions 进行，而不是 OpenClaw 的 `agentTurn`。有关重写模式，请参见 `skills/conventions/cron-via-minions.md`。v0.11.0 迁移自动重写处理程序为 gbrain 内置的条目；特定于主机的处理程序（例如 `ea-inbox-sweep`）需要根据 `docs/guides/plugin-handlers.md` 进行代码级注册。

## 架构

如何构建你的系统。

| 指南 | 涵盖内容 |
|-------|---------------|
| [双仓库架构](guides/repo-architecture.md) | 代理仓库 vs 大脑仓库，边界规则，决策树 |
| [子代理模型路由](guides/sub-agent-routing.md) | 哪个模型用于哪个任务，信号检测器模式，成本优化 |
| [三种搜索模式](guides/search-modes.md) | 关键词、混合、直接。何时使用每种 |
| [大脑 vs 代理记忆](guides/brain-vs-memory.md) | 3 层：GBrain（世界知识）、代理记忆、会话 |

## 集成

连接你的生活。

| 指南 | 涵盖内容 |
|-------|---------------|
| [凭证网关](integrations/credential-gateway.md) | ClawVisor / Hermes 用于 Gmail、日历、联系人 |
| [会议与通话 Webhook](integrations/meeting-webhooks.md) | Circleback 记录 + Quo/OpenPhone SMS/通话 |
| [语音到大脑](../recipes/twilio-voice-brain.md) | 电话 + WebRTC 浏览器通话创建大脑页面。25 个生产模式：身份分离、出价系统、对话计时、主动顾问、提示压缩、呼叫者路由、动态 VAD、实时日志记录、双重保障通话后处理 |
| [电子邮件到大脑](../recipes/email-to-brain.md) | Gmail 消息通过确定性收集器流入实体页面 |
| [X 到大脑](../recipes/x-to-brain.md) | Twitter 监控，带删除检测 + 参与速度 |
| [日历到大脑](../recipes/calendar-to-brain.md) | Google 日历事件成为可搜索的每日大脑页面 |
| [会议同步](../recipes/meeting-sync.md) | Circleback 记录自动导入，带与会者传播 |

## 管理

保持运行和更新。

| 指南 | 涵盖内容 |
|-------|---------------|
| [升级与自动更新](guides/upgrades-auto-update.md) | check-update、代理通知、迁移文件 |
| [实时同步](guides/live-sync.md) | 保持索引最新：cron、--watch、webhook 方法 |

## 入门

设置后，大脑是空的。冷启动技能按最高杠杆数据源顺序填充它：

| 指南 | 涵盖内容 |
|-------|---------------|
| [冷启动](../skills/cold-start/SKILL.md) | 第一天引导：联系人、日历、电子邮件、对话、社交、档案。使用 ClawVisor 进行安全凭证处理 —— 代理永远不持有原始 API 密钥。 |
| [询问用户](../skills/ask-user/SKILL.md) | 决策点人类输入的选择门模式。冷启动和其他技能使用。 |

---

## 附录：GBrain CLI 快速参考

| 命令 | 用途 |
|---------|---------|
| `gbrain search "term"` | 在所有大脑页面中搜索关键词 |
| `gbrain query "question"` | 混合搜索（向量 + 关键词 + RRF） |
| `gbrain get <slug>` | 通过 slug 读取特定大脑页面 |
| `gbrain sync` | 将本地 markdown 仓库同步到 gbrain 索引 |
| `gbrain import <path>` | 将文件导入大脑 |
| `gbrain embed --stale` | 重新嵌入过期或缺失嵌入的页面 |
| `gbrain integrations` | 管理集成方案（感官 + 反射） |
| `gbrain stats` | 显示大脑统计信息（页面计数、上次同步等） |
| `gbrain doctor` | 诊断大脑健康问题 |
| `gbrain check-update` | 检查新版本和集成方案 |

运行 `gbrain --help` 获取完整命令参考。

---

## 架构与哲学

- [基础设施层](architecture/infra-layer.md) — 导入管道、分块、嵌入、搜索
- [薄线束，厚技能](ethos/THIN_HARNESS_FAT_SKILLS.md) — 架构哲学
- [Markdown 技能作为配方](ethos/MARKDOWN_SKILLS_AS_RECIPES.md) — 为什么 markdown 是代码，你的代理是包管理器
- [个人 AI 的 Homebrew](designs/HOMEBREW_FOR_PERSONAL_AI.md) — 10 星愿景
- [推荐架构](GBRAIN_RECOMMENDED_SCHEMA.md) — 大脑仓库的目录结构
- [验证手册](GBRAIN_VERIFY.md) — 端到端安装验证