# 大脑 vs 记忆 vs 会话

## 目标
明确什么内容应该存放在 GBrain 中、什么应该存放在代理记忆中、什么应该保留在会话上下文中 —— 确保每一条信息都存储在正确的层级。

## 用户获得的价值

没有这个模式：人物档案被存储在代理记忆中（代理重置时丢失），用户偏好被存储在 GBrain 中（扰乱知识页面），代理会重复询问已经知道答案的问题。

有了这个模式：世界知识持久保存在大脑中，操作状态持久保存在代理记忆中，代理永远不会把信息放错层级。

## 实现

```
on new_information(info):
    # 三个层级，三个用途 —— 路由到正确的层级

    if info.is_about_the_world:
        # GBRAIN: 人物、公司、交易、会议、概念、想法
        # 这是世界知识 —— 关于代理外部实体的事实
        gbrain put <slug> --content "..."
        # 示例：
        #   "Pedro 是 Brex 的 CEO"           -> gbrain (人物页面)
        #   "Brex 在 D 轮融资中估值达 120 亿美元"   -> gbrain (公司页面)
        #   "周二会议讨论了 Q2 内容"   -> gbrain (会议页面)
        #   "肉身维护税概念"   -> gbrain (原创页面)

    elif info.is_about_operations:
        # AGENT MEMORY: 偏好、决策、工具配置、会话连续性
        # 这是代理的操作方式 —— 不是关于世界的事实
        memory_write(info)
        # 示例：
        #   "用户喜欢简洁的格式"      -> 代理记忆
        #   "先部署到 staging 再到 prod"        -> 代理记忆
        #   "代码块使用暗色主题"         -> 代理记忆
        #   "Crustdata 的 API 密钥放在 .env 中"   -> 代理记忆

    elif info.is_current_conversation:
        # SESSION CONTEXT: 刚刚说过的话、当前任务、即时状态
        # 这是自动的 —— 已经在对话窗口中
        # 无需存储操作
        # 示例：
        #   "我们刚才讨论了董事会演示文稿"  -> 会话
        #   "你让我审核这个 PR"          -> 会话
        #   "我刚刚分享的文件"                  -> 会话

# 查询路由：
on user_asks(question):
    if question.about_person or question.about_company or question.about_meeting:
        gbrain search "{entity}"    # -> 世界知识
        gbrain get <slug>

    elif question.about_preference or question.about_how_to_operate:
        memory_search("{topic}")    # -> 操作状态

    elif question.about_current_context:
        # 已经在会话中 —— 只需引用对话历史
        pass
```

## 需要注意的地方

1. **不要在代理记忆中存储人物信息。** "Pedro 更喜欢邮件而不是 Slack" 感觉像是一个偏好，但这是关于 Pedro 的事实 —— 应该放在 GBrain 的 Pedro 页面中。代理记忆是用于代理自己的操作状态，而不是关于世界上人物的事实。
2. **不要在 GBrain 中存储用户偏好。** "用户喜欢项目符号而不是段落" 是关于代理应该如何表现的，而不是关于世界的。它应该放在代理记忆中。GBrain 页面是用于实体的，而不是用于代理配置的。
3. **外部想法的综合应该放在 GBrain 中。** "用户对 Peter Thiel 的从 0 到 1 框架的看法" 是用户的原创思考 —— 应该放在 GBrain 的 originals/ 下，而不是代理记忆中。
4. **代理记忆在某些平台上无法在代理重置后存活。** 关键的世界知识必须在 GBrain 中，这是持久的。如果代理失去记忆，大脑仍然保存着一切。
5. **不确定时，问自己：这是关于世界还是关于如何操作？** 世界知识 -> GBrain。操作状态 -> 代理记忆。当前对话 -> 会话。

## 验证方法

1. 问代理 "Pedro 是谁？" —— 确认它运行 `gbrain search` 或 `gbrain get`，而不是 `memory_search`。人物查询应该访问 GBrain。
2. 问代理 "我应该如何格式化响应？" —— 确认它检查代理记忆，而不是 GBrain。偏好是操作状态。
3. 检查代理记忆存储中是否不存在人物或公司页面。运行 `memory_search "person"` —— 应该返回偏好，而不是档案。
4. 检查 GBrain 是否不包含关于代理行为的页面。运行 `gbrain search "user prefers"` —— 应该返回空（偏好属于代理记忆）。
5. 代理重置后，确认 GBrain 知识仍然可访问。运行 `gbrain get <any_slug>` —— 世界知识应该在重置后仍然存在。

---
*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*