# 安静时段和时区感知交付

## 目标

在睡眠时间内暂停所有通知，将暂停的消息合并到早晨简报中，并在用户旅行时自动调整。

## 用户获得的价值

没有这个功能：凌晨 3 点收到 cron 任务的通知。一次糟糕的通知就会让用户禁用整个系统。

有了这个功能：大脑在夜间工作（梦境周期、收集器、丰富化），但通知会暂停到早晨。去东京旅行？系统会从你的日历自动调整，无需更改配置。

## 实现方案

### 安静时段门控

每个发送通知的 cron 任务必须首先检查安静时段。

```
QUIET_START = 23  // 当地时间晚上 11 点
QUIET_END = 8     // 当地时间早上 8 点

is_quiet(local_hour):
  return local_hour >= QUIET_START OR local_hour < QUIET_END
```

**发送任何通知之前：**
1. 确定用户当前的时区（从配置或心跳状态）
2. 将当前 UTC 时间转换为当地时间
3. 如果是安静时段：暂停消息，不发送

### 暂停的消息

在安静时段，输出发送到暂停目录而不是直接发送：

```
if is_quiet():
  mkdir -p /tmp/cron-held/
  write("/tmp/cron-held/{job-name}.md", output)
  exit  // don't send
else:
  send(output)
```

早晨简报会拾取暂停的消息：

```
morning_briefing():
  held_files = list("/tmp/cron-held/*.md")
  if held_files:
    briefing += "## Overnight Updates\n\n"
    for file in held_files:
      briefing += read(file)
      delete(file)
```

这样不会丢失任何内容。夜间 cron 结果会被合并到用户早上看到的第一件事。

### 时区感知

代理应该知道用户所在的时区。将其存储在代理的操作状态中：

```json
{
  "currentLocation": {
    "timezone": "US/Pacific",
    "city": "San Francisco"
  }
}
```

**在以下情况更新时区：**
- 日历显示用户正在飞往某处（检查航空公司/酒店事件）
- 用户提到在不同的城市
- 用户的活动时间发生变化（他们在太平洋时间凌晨 3 点回复 = 他们可能在旅行）

**显示给用户的所有时间都应该是他们的本地时区。** 永远不要显示 UTC 或用户不在的时区。

### Shell 实现

```bash
#!/bin/bash
# quiet-hours-gate.sh — 在任何通知之前运行

TIMEZONE="${USER_TIMEZONE:-US/Pacific}"
LOCAL_HOUR=$(TZ="$TIMEZONE" date +%H)

if [ "$LOCAL_HOUR" -ge 23 ] || [ "$LOCAL_HOUR" -lt 8 ]; then
  echo "QUIET_HOURS=true"
  exit 1  # don't send
fi

echo "QUIET_HOURS=false"
exit 0  # ok to send
```

**在 cron 任务脚本中：**
```bash
# 首先检查安静时段
if ! bash scripts/quiet-hours-gate.sh; then
  mkdir -p /tmp/cron-held
  echo "$OUTPUT" > /tmp/cron-held/$(basename "$0" .sh).md
  exit 0
fi

# 不是安静时段 — 正常发送
send_notification "$OUTPUT"
```

### 可配置时段

有些用户想要不同的安静时段。存储配置：

```json
{
  "quiet_hours": {
    "start": 23,
    "end": 8,
    "enabled": true
  }
}
```

设置 `enabled: false` 以完全禁用安静时段（例如，用于 24/7 监控）。

## 难点

1. **每个任务都要门控。** 安静时段检查必须在每个产生通知的 cron 任务之前运行。即使有一个任务跳过了门控，用户也会在凌晨 3 点收到通知，并失去对整个系统的信任。无一例外。

2. **暂停的消息必须被拾取。** 如果早晨简报不读取 `/tmp/cron-held/`，夜间结果会静默消失。验证简报技能读取并清除暂停目录。孤立的暂停文件意味着拾取集成已损坏。

3. **时区自动检测很脆弱。** 基于日历的时区检测依赖于用户有带位置数据的航空公司/酒店事件。如果用户预订旅行但没有日历条目，系统将无法检测到移动。回退到活动时间分析（在太平洋时间凌晨 3 点回复 = 可能不再在太平洋时区），如果不确定则询问用户。

## 验证方法

1. **将安静时段设置为当前时段。** 临时将 `QUIET_START` 设置为现在之前一小时，`QUIET_END` 设置为现在之后一小时。触发 cron 任务。验证输出发送到 `/tmp/cron-held/` 而不是发送。

2. **检查暂停消息的拾取。** 在步骤 1 之后，运行或模拟早晨简报。验证暂停消息出现在 "Overnight Updates" 部分，并且文件从 `/tmp/cron-held/` 中删除。

3. **验证时区调整。** 将时区配置更改为当前是安静时段的时区。触发通知。验证它被暂停。在活动时间改回真实时区。再次触发。验证它发送成功。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*