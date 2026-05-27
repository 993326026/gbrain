# 插件开发者指南（v0.15）

`gbrain` 通过 `GBRAIN_PLUGIN_PATH` 从外部发现子代理定义。如果你维护下游代理（你的OpenClaw部署、工作流主机、私有工具）并想随其一起发布自定义子代理，只需在该环境变量指定的路径下放置一个插件目录即可。

本指南面向插件开发者。CLI用户无需阅读。

## 最小可行插件

```
/path/to/my-plugin/
├── gbrain.plugin.json
└── subagents/
    └── my-summarizer.md
```

`gbrain.plugin.json`:

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "plugin_version": "gbrain-plugin-v1"
}
```

`subagents/my-summarizer.md`:

```markdown
---
name: my-summarizer
model: claude-sonnet-4-6
allowed_tools:
  - brain_search
  - brain_get_page
---

你是一个大脑页面摘要器。给定一个slug，获取页面并生成三句话摘要。
```

## 启用插件

```bash
export GBRAIN_PLUGIN_PATH="/path/to/my-plugin"
gbrain jobs work           # 工作进程启动时会打印插件加载行
gbrain agent run "summarize meetings/2026-04-20" --subagent-def my-summarizer
```

多个插件：用冒号分隔，就像 `$PATH` 一样。

```bash
export GBRAIN_PLUGIN_PATH="/path/to/plugin-a:/path/to/plugin-b"
```

## 规则（设计上严格）

**路径策略**。只允许绝对路径。相对路径、`~` 前缀路径和URL格式路径（`https://`、`file://`）会被拒绝并发出警告。你控制插件在磁盘上的位置；`gbrain` 不会猜测。

**冲突策略**。如果两个插件发布同名子代理，`GBRAIN_PLUGIN_PATH` 中列出的第一个获胜。另一个会被丢弃并发出警告，同时命名两个来源。

**信任策略**。v0.15中插件只能发布子代理定义：

- 你**不能**声明新工具。
- 你**不能**扩展大脑工具白名单。
- 你**不能**覆盖任何 `agentSafe` 或类似标志。
- 你的 `allowed_tools:` 前置字段**必须**是派生大脑工具注册表的子集。不在注册表中的名称会在插件加载时（工作进程启动时）被拒绝，而不是在子代理调度时——因此插件中的拼写错误会在启动时产生响亮的错误，而不是在凌晨3点静默地"工具从不触发"。

v0.16+ 可能会通过单独的合约开放插件声明的工具。不要期望它很快实现。

## `gbrain.plugin.json`

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 人类可读的插件ID。出现在警告和冲突日志中。 |
| `version` | string | 是 | 你的插件的semver版本。仅供参考。 |
| `plugin_version` | string | 是 | 合约锁定。v0.15必须等于 `"gbrain-plugin-v1"`。 |
| `subagents` | string | 否 | 子目录名称（默认为 `subagents`）。尝试规避会被拒绝。 |
| `description` | string | 否 | 显示在未来的 `gbrain plugin list` 中。 |

## 子代理定义文件

带有YAML前置的普通markdown。正文是系统提示。前置控制运行时行为。

识别的前置字段：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 否 | 子代理标识符，用作 `--subagent-def`。默认为文件基名。 |
| `model` | string | 否 | Anthropic模型ID。默认为处理器默认值（sonnet）。 |
| `max_turns` | number | 否 | 助理轮数上限。默认为20。 |
| `allowed_tools` | string[] | 否 | 工具名称白名单。必须是派生大脑注册表的子集。不匹配时被拒绝。 |

未知的前置字段会被保留但被处理器忽略。v0.16可能会使用更多字段。

## 会困扰你的注意事项

1. **插件定义在运行期间不能更改。** 加载器在工作进程启动时读取一次磁盘。编辑子代理定义不会重新生效，直到你重新启动工作进程。这是故意的——实时重载会破坏崩溃恢复重放。

2. **`~/.gbrain/audit/subagent-jobs-*.jsonl` 仅本地可用。** 如果你的工作进程在与 `gbrain agent logs` 调用者不同的主机上运行，CLI将看不到该工作进程的心跳。v0.16将统一这一点；目前假设工作进程 + CLI共享文件系统。

3. **工具调用始终以 `ctx.remote = true` 运行。** 即使在本地CLI调用时也是如此。依赖 `remote=true` 的工具（file_upload的严格限制、put_page的命名空间检查）将生效。这是良好的默认值；想要在大脑之外访问本地文件系统的子代理定义无法实现。

4. **`put_page` 写入受命名空间限制。** ID为42的子代理只能在 `wiki/agents/42/...` 下写入。这在工具架构（向模型显示的slug模式）和 `put_page` 操作的服务器端（如果 `viaSubagent=true` 则失败关闭）都有强制执行。不要试图绕过它；你会得到 `permission_denied`。

## 示例：下游OpenClaw插件

```
~/your-openclaw/
└── gbrain-plugin/
    ├── gbrain.plugin.json
    └── subagents/
        ├── meeting-ingestion.md
        ├── signal-detector.md
        └── daily-task-prep.md
```

`~/your-openclaw/gbrain-plugin/gbrain.plugin.json`:

```json
{
  "name": "your-openclaw",
  "version": "2026.4.20",
  "plugin_version": "gbrain-plugin-v1",
  "description": "Your OpenClaw's personal-brain subagents"
}
```

环境变量：

```bash
export GBRAIN_PLUGIN_PATH="$HOME/your-openclaw/gbrain-plugin"
```

然后你的OpenClaw调用 `gbrain agent run --subagent-def meeting-ingestion --fanout-by transcript ...`，其定义会自动加载。