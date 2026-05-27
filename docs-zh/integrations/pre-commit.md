# 大脑仓库的预提交钩子（v0.22.4+）

`gbrain frontmatter install-hook` 在你的大脑源仓库中安装一个 git 预提交钩子，对暂存的 `.md` 和 `.mdx` 文件运行 `gbrain frontmatter validate`。格式错误的 frontmatter 会阻止提交。使用 `git commit --no-verify` 绕过。

## 钩子捕获的内容

与 `frontmatter-guard` 技能和 `gbrain doctor` 的 `frontmatter_integrity` 子检查报告相同的七个验证类：

| 代码 | 捕获内容 |
|------|---------|
| `MISSING_OPEN` | 文件不以 `---` 开头 |
| `MISSING_CLOSE` | 第一个标题前没有闭合的 `---` |
| `YAML_PARSE` | YAML 解析失败（语法或结构） |
| `SLUG_MISMATCH` | frontmatter 中的 `slug:` 与路径派生的 slug 不匹配 |
| `NULL_BYTES` | 内容中任何位置的二进制损坏（`\x00`） |
| `NESTED_QUOTES` | `title: "outer "inner" outer"` 这种破坏 YAML 的形状 |
| `EMPTY_FRONTMATTER` | `---` ... `---` 之间没有有意义的内容 |

## 安装

对于所有已注册的 git 仓库源：

```bash
gbrain frontmatter install-hook
```

对于一个源：

```bash
gbrain frontmatter install-hook --source <id>
```

强制覆盖现有预提交钩子（写入 `.bak`）：

```bash
gbrain frontmatter install-hook --force
```

钩子位于 `<source>/.githooks/pre-commit`。如果 `core.hooksPath` 未设置，安装还会运行 `git config core.hooksPath .githooks`，因此无需手动 git 配置即可拾取钩子。

## 绕过

标准 git 逃生舱口：

```bash
git commit --no-verify
```

这会跳过所有预提交钩子。谨慎使用 — 用户下次运行 `gbrain doctor` 时，问题会浮出水面。

## 卸载

```bash
gbrain frontmatter install-hook --uninstall
```

如果安装期间保存了 `.bak`，它会被恢复为活动钩子。否则钩子会被干净地移除。

## 在未安装 gbrain 的机器上的行为

钩子脚本检查 `$PATH` 上的 `gbrain`。当缺失时，它向 stderr 打印一行警告并退出 0 — 不会仅仅因为开发者没有在本地安装 gbrain 就阻止提交。安装 gbrain 后，钩子恢复阻止格式错误的页面。

## 对于下游代理 fork

如果你的 OpenClaw 在非大脑仓库本身的主机仓库中包装 gbrain，你可能需要单独的钩子策略：

- **大脑仓库就是主机仓库**（gbrain 技能 + 大脑页面在一个仓库中）：按上述方式通过 `gbrain frontmatter install-hook` 安装。
- **大脑仓库是单独注册的源**（例如 `~/brain` 注册为源，主机仓库是 `~/agent-fork`）：仅在大脑仓库中安装；agent-fork 代码不需要此钩子。
- **大脑仓库是自动生成的**（例如由同步守护进程写入桶）：完全跳过钩子；改为通过 `import { writeBrainPage } from 'gbrain/brain-writer'` 在写入器处进行门控（计划在后续版本中实现；当前 CLI 是接口）。

## 它如何融入更广泛的 frontmatter 管道

```
agent writes a page         git commit                 doctor scan
       ↓                          ↓                          ↓
[source content]   →  [pre-commit hook validates]   →  [frontmatter_integrity check]
       ↓                          ↓                          ↓
  raw file on disk       blocks malformed commits     surfaces existing issues
                                                             ↓
                                                  `gbrain frontmatter validate
                                                   <source-path> --fix`
                                                   (writes .bak backups)
```

钩子是写入时门控；doctor 是审计门控；CLI 是修复工具。它们共享 `parseMarkdown(..., {validate:true})` 作为什么算作格式错误的单一事实来源。