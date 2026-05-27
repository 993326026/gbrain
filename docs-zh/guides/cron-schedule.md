# 定时任务参考

## 目标

生产环境的大脑运行20+个定期任务，使其保持活跃、更新和复合增长。本指南展示了时间表、模式以及如何设置。

## 用户获得的价值

没有这个：大脑仅在你手动摄取数据时更新。页面变得陈旧，实体变得单薄，引用断开，代理从旧上下文中回答。

有了这个：大脑自我维护。电子邮件、社交、日历和会议自动流入。单薄的页面在一夜之间变得丰富。断开的引用被修复。你醒来时大脑比你睡觉时更聪明。

## 时间表

| 频率 | 任务 | 大脑交互 | 配方 |
|------|------|----------|------|
| 每30分钟 | 邮件监控 | 搜索发件人，更新人物页面 | [email-to-brain](../../recipes/email-to-brain.md) |
| 每30分钟 | X/Twitter收集 | 创建/更新媒体页面，实体提取 | [x-to-brain](../../recipes/x-to-brain.md) |
| 每天3次（工作日） | 会议同步 | 完整摄取 + 参会者传播 | [meeting-sync](../../recipes/meeting-sync.md) |
| 每周 | 日历同步 | 每日文件 + 参会者丰富 | [calendar-to-brain](../../recipes/calendar-to-brain.md) |
| 每日上午 | 晨间简报 | 搜索日历参会者、交易状态、活跃线程 | [briefing skill](../../skills/briefing/SKILL.md) |
| 每周 | 大脑维护 | `gbrain doctor`，嵌入过期内容，孤立检测 | [maintain skill](../../skills/maintain/SKILL.md) |
| 每晚 | 梦境循环 | 实体扫描，丰富单薄点，修复引用 | 见下文 |

## 实现：设置定时任务

```bash
# 邮件收集器 —— 每30分钟
*/30 * * * * cd /path/to/email-collector && node email-collector.mjs collect && node email-collector.mjs digest

# X/Twitter收集器 —— 每30分钟
*/30 * * * * cd /path/to/x-collector && node x-collector.mjs collect >> /tmp/x-collector.log 2>&1

# 会议同步 —— 工作日上午10点、下午4点、晚上9点
0 10,16,21 * * 1-5 cd /path/to/meeting-sync && node meeting-sync.mjs >> /tmp/meeting-sync.log 2>&1

# 日历同步 —— 周日上午10点
0 10 * * 0 cd /path/to/calendar-sync && node calendar-sync.mjs --start $(date -v-7d +%Y-%m-%d) --end $(date +%Y-%m-%d)

# 大脑健康 —— 每周一上午6点
0 6 * * 1 gbrain doctor --json >> /tmp/gbrain-health.log 2>&1 && gbrain embed --stale

# 梦境循环 —— 每晚凌晨2点
0 2 * * * /path/to/dream-cycle.sh
```

### 安静时段门（强制性）

每个发送通知的定时任务必须首先检查安静时段。
请参阅 [安静时段](quiet-hours.md) 获取完整模式。

```bash
# 在每个定时脚本中：
if ! bash scripts/quiet-hours-gate.sh; then
  mkdir -p /tmp/cron-held
  echo "$OUTPUT" > /tmp/cron-held/$(basename "$0" .sh).md
  exit 0
fi
# 非安静时段 —— 正常发送
```

### 旅行感知时区处理

代理从你的日历读取航班、酒店和外出办公时段，
推断你当前的位置和时区。所有时间都显示在你的本地时区。

```
// 示例：用户飞往东京
// 太平洋时间下午2点 = 东京时间凌晨3点 = 安静时段
// 保持通知，合并到晨间简报

get_user_timezone():
  calendar = gbrain search "flight" --type calendar --recent 7d
  if recent_flight:
    return infer_timezone(flight.destination)
  return config.default_timezone  // 回退：US/Pacific
```

当你旅行时：在家中会在你清醒时间触发但在目的地会在你睡眠时间触发的定时任务会被保持并合并到下一次晨间简报中。无需任何配置更改。

## 梦境循环

最重要的定时任务。在你睡觉时运行。

### 它做什么

```
dream_cycle():
  // 阶段1：实体扫描
  conversations = get_todays_conversations()
  for message in conversations:
    entities = detect_entities(message)
    for entity in entities:
      page = gbrain search "{entity.name}"
      if not page:
        create_page(entity)        // 新实体，创建 + 丰富
      elif page.is_thin():
        enrich_page(entity)        // 单薄页面，填充它
      else:
        update_timeline(entity)    // 现有页面，添加今天的提及

  // 阶段2：修复断开的引用
  pages = gbrain list --type person --limit 100
  for page in pages:
    for entry in page.timeline:
      if not entry.has_source_attribution():
        fix_citation(entry)        // 在缺失的地方添加 [来源: ...]
      if entry.has_tweet_url() and not entry.url_is_valid():
        fix_url(entry)             // 断开的推文链接

  // 阶段3：整合记忆
  patterns = detect_patterns_across_conversations()
  for pattern in patterns:
    promote_to_memory(pattern)     // 临时 → 持久知识

  // 阶段4：同步
  gbrain sync --no-pull --no-embed
  gbrain embed --stale
```

### 设置梦境循环

**OpenClaw:** 附带DREAMS.md作为默认技能。三个阶段（浅、深、REM）在安静时段自动运行。

**Hermes Agent:**
```bash
/cron add "0 2 * * *" "梦境循环：搜索今天的会话中我提到的实体。
  对于每个人、公司或想法：检查大脑页面是否存在（gbrain search），
  如果单薄则创建或更新。修复任何断开的引用。然后整合：读取MEMORY.md，
  提升重要信号，删除陈旧条目。"
  --name "nightly-dream-cycle"
```

**Claude Code / 自定义代理:** 创建脚本：
```bash
#!/bin/bash
# dream-cycle.sh

# 检查安静时段（应该是安静的 —— 这是我们运行的时间）
echo "梦境循环开始于 $(date)"

# 阶段1：实体扫描（生成子代理）
# 读取今天的对话日志，提取实体，更新大脑

# 阶段2：引用卫生
gbrain doctor --json | jq '.checks[] | select(.status=="warn")'

# 阶段3：嵌入任何过期内容
gbrain embed --stale

echo "梦境循环完成于 $(date)"
```

## 需要注意的地方

1. **梦境循环不是可选的。** 没有它，信号会从每次对话中泄漏出去。有了它，什么都不会丢失。这是一个忘记和一个记住的代理之间的区别。

2. **每个通知任务都要有安静时段门。** 如果你跳过它，用户会在凌晨3点被ping。一次凌晨3点的ping，他们就会禁用整个系统。

3. **不要过度使用cron。** 20+个任务听起来很多。从以下开始：邮件（30分钟）、梦境循环（每晚）、大脑健康（每周）。添加更多集成配方时再添加更多任务。

4. **时区变化是自动的。** 不要让用户在旅行时重新配置cron。读取日历，推断时区，调整交付。

5. **保持的消息必须被拾取。** 如果安静时段保持了通知，晨间简报必须包含它。否则信息会丢失。

## 验证方法

1. **安静时段：** 将安静时段设置为当前小时。运行通知cron。验证输出转到`/tmp/cron-held/`，而不是消息传递。
2. **梦境循环：** 手动运行梦境循环。检查单薄的实体页面是否得到丰富，断开的引用是否被修复。
3. **邮件收集器cron：** 等待30分钟。检查`data/digests/`获取新摘要。
4. **晨间简报：** 检查保持的消息是否出现在简报中。
5. **健康检查：** 运行`gbrain doctor --json`。所有检查都应该通过。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。另见：[安静时段](quiet-hours.md)、[运营规范](operational-disciplines.md)*