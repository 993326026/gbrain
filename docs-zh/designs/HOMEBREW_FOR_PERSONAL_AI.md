# 个人 AI 基础设施的 Homebrew 方案

GBrain 集成系统的 10 星级愿景。发布 Approach B（v0.7.0），在后续版本中逐步构建实现。

## 愿景

GBrain 成为个人基础设施操作系统，您生活中的每个信号自动流经大脑。集成是**感官**（数据输入）和**反射**（对模式的自动响应）。用户订阅创造者的实际操作系统，然后自定义它。

```
$ gbrain integrations

  SENSES (data inputs)                          STATUS
  -------------------------------------------------------
  voice-to-brain    Phone calls -> brain pages  ACTIVE    last call: 2h ago
  email-to-brain    Gmail -> entity updates     ACTIVE    47 emails today
  x-to-brain        Twitter -> media pages      ACTIVE    312 tweets tracked
  calendar-to-brain Google Cal -> meeting prep  ACTIVE    3 meetings tomorrow
  photos-to-brain   Camera roll -> visual mem   AVAILABLE
  slack-to-brain    Slack -> conversation index  AVAILABLE
  rss-to-brain      RSS feeds -> media pages     AVAILABLE

  REFLEXES (automated responses)                STATUS
  -------------------------------------------------------
  meeting-prep      Brief me before meetings    ACTIVE    next: 9am tomorrow
  entity-enrich     Auto-enrich new contacts    ACTIVE    12 enriched today
  dream-cycle       Overnight brain maintenance ACTIVE    last run: 3am
  deal-tracker      Alert on deal changes       AVAILABLE
  follow-up-nudge   Remind on stale threads     AVAILABLE

  This week: 1,247 signals ingested. Top: email (47%), voice (23%), X (18%).
  34 new entity pages created. 7 calls transcribed.

  Run 'gbrain integrations show <id>' for setup details.
```

用户感觉："我的大脑是活的。它在关注我关心的一切，每天都在变得更聪明。我不必编写任何代码。我只是在代理问的时候说'是'。"

## 架构：感官 & 反射

### Recipe 格式（YAML frontmatter + markdown body）

```yaml
---
id: voice-to-brain
name: Voice-to-Brain
version: 0.7.0
description: Phone calls create brain pages via Twilio + OpenAI Realtime + GBrain MCP
category: sense
requires: [credential-gateway]
secrets:
  - name: TWILIO_ACCOUNT_SID
    description: Twilio account SID
    where: https://console.twilio.com
  - name: OPENAI_API_KEY
    description: OpenAI API key (for Realtime voice)
    where: https://platform.openai.com/api-keys
health_checks:
  - curl -s https://api.twilio.com/2010-04-01 > /dev/null
  - curl -s https://api.openai.com/v1/models > /dev/null
setup_time: 30 min
---

[Opinionated setup instructions the agent executes...]
```

### 依赖图

Recipes 在 frontmatter 中声明 `requires`。CLI 在设置前解析依赖项。如果 voice-to-brain 需要 credential-gateway，代理首先设置 credential-gateway。

```
credential-gateway
  ├── voice-to-brain (requires credentials for Twilio)
  ├── email-to-brain (requires credentials for Gmail)
  └── calendar-to-brain (requires credentials for Google Calendar)

x-to-brain (standalone, uses X API directly)
```

### 健康仪表板

`gbrain integrations doctor` 运行每个已配置 recipe 的 health_checks：
```
$ gbrain integrations doctor
  voice-to-brain:   ✓ Twilio reachable  ✓ OpenAI key valid  ✓ ngrok tunnel up
  email-to-brain:   ✓ Gmail auth valid   ✗ No emails in 48h (check cron)
  OVERALL: 1 warning
```

### 感官分析

`gbrain integrations stats` 聚合心跳数据：
```
$ gbrain integrations stats
  This week: 1,247 signals ingested
  Top sources: email (47%), voice (23%), X (18%), calendar (12%)
  34 new entity pages created
  7 calls transcribed
  Brain growth: 12,400 → 12,834 pages (+434)
```

### 反射规则引擎（未来）

反射是在大脑状态变化时触发的 recipes：

```yaml
---
id: deal-tracker
category: reflex
triggers:
  - type: page_updated
    filter: {type: deal, field: status}
  - type: timeline_entry
    filter: {source: email, mentions: deal}
action: alert
---

When a deal page's status changes or a new email mentions a deal,
alert the user with context from the brain.
```

## 路线图

| 版本 | 发布内容 | 关键 Recipe |
|---------|-----------|------------|
| v0.7.0 | Recipe 格式、CLI、SKILLPACK 突破 | voice-to-brain |
| v0.8.0 | 3 个更多感官、反射格式 | email, X, calendar |
| v0.9.0 | 社区 recipes、安装执行器 | community submissions |
| v1.0.0 | 完整感官/反射、健康仪表板 | meeting-prep, dream-cycle |

## 关键设计决策

1. **GBrain 是确定性基础设施。** 跨感官关联、模式检测和智能响应是代理的工作（OpenClaw/Hermes）。GBrain 提供管道。

2. **代理就是运行时。** 没有 npm 包、Docker 镜像或确定性脚本。recipe markdown 就是安装程序。代理读取它并完成工作。

3. **非常固执己见的默认值。** 将创造者的确切生产设置作为默认值发布。用户从那里自定义。未知来电者会被筛选。安静时间被强制执行。每次通话都会进行大脑优先查找。

4. **代理可读输出。** 所有 CLI 输出必须可被代理解析（--json 标志）。迁移文件包含代理指令。代理是主要消费者，而不是人类。