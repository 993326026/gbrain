# 升级和自动更新通知

## 目标

用户以对话方式收到新 GBrain 功能的通知，代理引导他们完成升级，包括升级后迁移，使新版本真正工作。

## 用户获得的价值

没有这个：GBrain 发布更新但没人知道。用户停留在旧版本，技能过时，功能缺失。更糟的是，有人运行 `gbrain upgrade` 但跳过升级后步骤，导致新代码与旧代理行为不兼容。

有了这个：代理每天检查更新，用简洁的利益导向要点推销升级，等待明确许可，然后运行完整的升级流程，包括重新读取技能、运行迁移和同步架构。用户自动获得新功能。

## 实现方案

### 检查（cron 启动）

```
check_for_update():
  result = run("gbrain check-update --json")

  if not result.update_available:
    exit_silently()  // 不要向用户发送消息

  // 推销升级 — 突出他们能做什么，而不是什么改变了
  message = compose_upgrade_message(
    current: result.current_version,
    latest: result.latest_version,
    changelog: result.changelog
  )
  send_to_user(message, respect_quiet_hours=true)
```

### 升级消息

推销升级。用户应该感觉"太好了，我想要"。突出他们现在能做以前不能做的事情，而不是文件变更。

```
> **GBrain v0.5.0 已可用**（你当前使用 v0.4.0）
>
> 新增功能：
> - 你的大脑永不过时。实时同步自动保持向量数据库最新，
>   因此编辑在几分钟内显示在搜索结果中
> - 新的验证手册在静默故障影响你之前捕获它们
> - 新安装自动设置实时同步。不再需要手动设置步骤
>
> 要我升级吗？我会更新所有内容并刷新我的操作手册。
>
> （回复 **yes** 升级，**not now** 跳过，**weekly** 减少检查频率，
> 或 **stop** 关闭更新检查）
```

### 处理响应

| 用户说 | 操作 |
|--------|------|
| yes / y / sure / ok / do it / upgrade | 运行完整升级流程（如下） |
| not now / later / skip / snooze | 确认，下次周期再检查 |
| weekly | 存储偏好，切换 cron 到每周 |
| daily | 存储偏好，切换 cron 回每日 |
| stop / unsubscribe / no more | 禁用 cron。告诉用户如何恢复 |

**切勿自动升级。** 始终等待明确确认。

### 完整升级流程（用户说 yes 后）

```
full_upgrade():
  // 步骤1：更新二进制/包
  run("gbrain upgrade")

  // 步骤2：重新读取所有更新的技能
  for skill in find("skills/*/SKILL.md"):
    read_and_internalize(skill)  // 更新的技能 = 更好的代理行为

  // 步骤3：重新读取生产参考文档
  read("docs/GBRAIN_SKILLPACK.md")
  read("docs/GBRAIN_RECOMMENDED_SCHEMA.md")

  // 步骤4：检查版本特定的迁移指令
  for version in range(old_version, new_version):
    migration = find(f"skills/migrations/v{version}.md")
    if migration exists:
      read_and_execute(migration)  // 按顺序执行，不要跳过

  // 步骤5：架构同步 — 建议新项，尊重拒绝项
  state = read("~/.gbrain/update-state.json")
  for recommendation in new_schema_recommendations:
    if recommendation not in state.declined:
      suggest_to_user(recommendation)
  update(state, new_choices)

  // 步骤6：报告变更
  summarize_to_user(actions_taken)
```

### 迁移文件

迁移文件位于 `skills/migrations/vX.Y.Z.md`。它们包含代理指令（不是脚本），用于使新版本对现有用户生效的升级后操作。例如：v0.5.0 迁移设置实时同步并运行验证手册。

代理按版本顺序读取迁移文件并逐步执行。没有迁移，代理有新代码但用户环境没有改变。

### Cron 注册

```
Name: gbrain-update-check
Default schedule: 0 9 * * * (每日 上午9点)
Weekly schedule: 0 9 * * 1 (周一 上午9点)
Prompt: "Run gbrain check-update --json. 如果 update_available 为 true，
  总结更新日志并发消息询问我是否要升级。
  如果为 false，保持静默。"
```

### 频率偏好

默认：每日。存储在代理内存中为 `gbrain_update_frequency: daily|weekly|off`。也持久化在 `~/.gbrain/update-state.json` 中，使其在代理上下文重置后仍然有效。

### 独立 Skillpack 用户

如果你直接加载此 SKILLPACK（从 GitHub 复制或读取）而没有安装 gbrain，你仍然可以保持最新。GBRAIN_SKILLPACK.md 和 GBRAIN_RECOMMENDED_SCHEMA.md 都有版本标记：

```bash
curl -s https://raw.githubusercontent.com/garrytan/gbrain/master/docs/GBRAIN_SKILLPACK.md | head -1
# 返回: <!-- skillpack-version: X.Y.Z -->
```

如果远程版本更新，请获取完整文件并替换本地副本。设置每周 cron 自动检查。

## 难点

1. **切勿自动安装。** 升级必须始终等待用户明确的"yes"。即使 cron 在上午9点检测到更新且更新日志看起来很棒，代理也会向用户发送消息并等待。自动安装可能破坏工作流、引入破坏性变更或中断正在进行的工作。

2. **迁移文件是代理指令，不是脚本。** 它们用简单语言逐步告诉代理做什么。它们**不是**盲目执行的 bash 脚本。代理读取它们，理解上下文，并适应用户的特定环境（例如，如果用户已经配置了实时同步，则跳过步骤）。

3. **check-update 应该在每日 cron 上运行。** 不要依赖用户记得检查更新。cron 每天上午9点运行 `gbrain check-update --json`（遵守安静时段）。如果没有新内容，完全保持静默。用户只在有值得升级的内容时才会收到更新通知。

## 验证方法

1. **运行 check-update 并验证检测。** 执行 `gbrain check-update --json`。验证它返回当前版本并正确报告是否有更新可用。如果 `update_available` 为 false，验证版本与 GitHub 上的最新发布匹配。

2. **验证迁移文件可读。** 列出 `skills/migrations/` 并检查每个文件是否遵循命名约定 `vX.Y.Z.md`。打开一个文件并验证它包含逐步的代理指令，而不是原始脚本。代理应该能够读取并执行每个步骤。

3. **端到端测试完整升级流程。** 如果有更新可用，说"yes"并观察代理执行完整流程：升级、重新读取技能、运行迁移、同步架构、报告。验证每个步骤完成且代理报告变更。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*