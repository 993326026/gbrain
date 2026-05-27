# 升级下游代理

GBrain 在 `skills/` 中提供技能。下游代理（自定义 OpenClaw 部署、任何类型的代理 fork）通常**复制**这些技能文件到自己的工作区，并随着时间推移产生分歧 —— 添加代理特定的阶段、删除不相关的内容、收紧语言。一旦发生这种情况，gbrain 就无法向这些 fork 推送更新。代理必须手动应用差异。

本文档列出了每个下游代理在升级时需要应用的确切差异。对照你 fork 的本地技能文件进行交叉引用。

## 为什么存在

`gbrain upgrade` 发布新的二进制文件。`gbrain post-upgrade [--execute --yes]` 运行架构迁移并回填数据。但是**技能文件本身**（告诉代理如何行为的文件）是用户拥有的。如果你的 `~/git/<your-agent>/workspace/skills/brain-ops/SKILL.md` 在顶部说 `# Based on gbrain v0.10.0`，它就不知道 v0.12.0 的功能。

代理会继续在每次 `put_page` 后手动调用 `gbrain link`（现在是冗余的 —— auto-link 会自动处理），错过用于关系问题的 `gbrain graph-query`，并且不知道回填结构化时间线。

## 如何应用

1. 识别你 fork 的技能文件。通常在 `~/git/<your-agent>/workspace/skills/` 或代理技能目录所在的任何位置。
2. 对于下面列出的每个技能，在你的 fork 中找到匹配的阶段/部分。
3. 应用差异（在指定位置粘贴新块）。
4. 更新 fork 顶部的版本横幅（`# Based on gbrain v0.12.0`）。
5. 验证：让代理编写一个测试页面，并确认响应包含 `auto_links: { created, removed, errors }`。

总时间：所有四个技能约 10 分钟。

---

## 1. brain-ops/SKILL.md

**位置：** 在 `### Phase 2: On Every Inbound Signal` 之后立即插入一个新的 `### Phase 2.5` 部分。

**原因：** Phase 2.5 声明 auto-link 自动运行。否则，代理的心理模型认为必须在每次 `put_page` 后调用 `gbrain link`，这现在是冗余的，可能导致重复添加警告。

```markdown
### Phase 2.5: Structured Graph Updates (automatic)

Every `put_page` call automatically extracts entity references and writes them
to the graph (`links` table) with inferred relationship types. Stale links
(refs no longer in the page text) are removed in the same call. This is
"auto-link" reconciliation.

- No manual `add_link` calls needed for ordinary page writes.
- Inferred link types: `attended` (meeting -> person), `works_at`, `invested_in`,
  `founded`, `advises`, `source` (frontmatter), `mentions` (default).
- The `put_page` MCP response includes `auto_links: { created, removed, errors }`
  so the agent can verify outcomes.
- To disable: `gbrain config set auto_link false`. Default is on.
- Timeline entries with specific dates still need explicit `gbrain timeline-add`
  (or batch via `gbrain extract timeline --source db`).
```

**同时更新铁律部分。** 如果你的 fork 仍然说 "Back-links maintained on every brain write (Iron Law)" 没有限定条件，追加：

```markdown
**v0.12.0 update:** Auto-link satisfies the Iron Law for entity-reference links
on every `put_page`. The agent's Iron Law obligation is now: include the
entity reference in the page content (e.g., `[Alice](people/alice)`); auto-link
handles the structured row. Manual `add_link` calls are reserved for
relationships you can't express in markdown content.
```

---

## 2. meeting-ingestion/SKILL.md

**位置：** 追加到 `### Phase 3: Attendee enrichment` 的末尾。

**原因：** 消除每位与会者的冗余 `gbrain link` 调用（当会议页面将与会者引用为 `[Name](people/slug)` 时，auto-link 会处理它们）。

```markdown
**Note (v0.12.0):** Once the meeting page is written via `gbrain put`, the
auto-link post-hook automatically creates `attended` links from the meeting
to each attendee whose page is referenced as `[Name](people/slug)`. You don't
need to call `gbrain link` for attendees. You DO still need `gbrain timeline-add`
for dated events (auto-link only handles links, not timeline entries).
```

**位置：** 在 `### Phase 4: Entity propagation` 中，"Back-link from entity page to meeting page" 这一行可以替换为：

```markdown
4. Entity references in the meeting page body auto-create the link via auto-link.
   For incoming references on the entity page (entity page → meeting page), edit
   the entity page to mention the meeting and `put_page` it — auto-link handles
   the rest.
```

---

## 3. signal-detector/SKILL.md

**位置：** 追加到 `### Phase 2: Entity Detection` 的末尾。

**原因：** 与 brain-ops 相同的逻辑 —— 消除在编写引用人物或公司的 originals/ideas 页面后手动调用 `gbrain link`。

```markdown
**Auto-link (v0.12.0):** When you write/update an originals or ideas page that
references a person or company, the auto-link post-hook on `put_page`
automatically creates the link from the new page to that entity. You don't
need to call `gbrain link` manually. Timeline entries still need explicit calls.
```

---

## 4. enrich/SKILL.md

**位置：** 用 v0.12.0 版本替换 `### Step 7: Cross-reference`。

**原因：** Step 7 以前主要是关于在相关实体页面之间创建链接。有了 auto-link，这是自动的。Step 7 现在是关于内容更新，而不是链接创建。

旧版本（删除）：
```markdown
### Step 7: Cross-reference

- Update company pages from person enrichment (and vice versa)
- Update related project/deal pages if relevant context surfaced
- Check index files if the brain uses them
- Add back-links manually via `gbrain link` for any new entity references
```

新版本（粘贴）：
```markdown
### Step 7: Cross-reference

- Update company pages from person enrichment (and vice versa)
- Update related project/deal pages if relevant context surfaced
- Check index files if the brain uses them

**Note (v0.12.0):** Links between brain pages are auto-created on every
`put_page` call (auto-link post-hook). Step 7 focuses on content
cross-references (updating related pages' compiled truth with new signal
from this enrichment), not on creating links. Verify via the `auto_links`
field in the put_page response (`{ created, removed, errors }`).
Timeline entries still need explicit `gbrain timeline-add` calls.
```

---

## 应用所有四个差异后

1. **提升版本横幅** 在每个 fork 文件的顶部：
   ```
   # Based on gbrain v0.12.0 skills/<skill-name>, extended with <your-agent>-specific config
   ```

2. **运行 v0.12.0 回填**（这会为您现有的大脑填充图）：
   ```bash
   gbrain post-upgrade
   ```
   v0.12.0 版本将 post-upgrade 连接到自动调用 `apply-migrations --yes`，这会运行 v0_12_0 编排器（架构 → 配置检查 → `extract links --source db` → `extract timeline --source db` → 验证）。幂等操作；当没有待处理内容时成本很低。

3. **验证 auto-link 工作：** 让代理编写一个引用 `[Some Person](people/some-person)` 的测试页面。确认 put_page 响应包含 `auto_links: { created: 1, removed: 0, errors: 0 }`。

4. **验证图遍历工作：**
   ```bash
   gbrain graph-query people/some-well-connected-person --depth 2
   ```
   应该返回类型化边的缩进树。

---

## v0.12.2 热修复（数据正确性，无需技能编辑）

v0.12.2 是一个 Postgres 数据正确性热修复。不需要更改 fork 的技能文件 —— 技能契约不变。但是您**确实**需要运行迁移，并且应该了解 markdown 解析中的一个行为变化。

### 1. 运行迁移（Postgres 支持的大脑）

```bash
gbrain upgrade
```

`v0_12_2` 编排器自动运行 `gbrain repair-jsonb`。它重写 `jsonb_typeof = 'string'` 的行，涉及 `pages.frontmatter`、`raw_data.data`、`ingest_log.pages_updated`、`files.metadata` 和 `page_versions.frontmatter`。幂等操作，安全重新运行。PGLite 大脑干净地执行空操作。

升级后验证：

```bash
gbrain repair-jsonb --dry-run --json    # expect totalRepaired: 0
```

### 2. 恢复任何被截断的 wiki 文章

如果您的大脑在 v0.12.2 之前导入了 wiki 风格的 markdown，某些页面可能被静默截断（正文中的任何独立 `---` 被视为时间线分隔符）。从源重新导入：

```bash
gbrain sync --full
```

新的 `splitBody` 正确重建 `compiled_truth`。

### 3. 了解未来的 splitBody 契约

`splitBody` 现在需要显式的时间线标记。识别的标记（优先级顺序）：

1. `<!-- timeline -->`（首选 —— `serializeMarkdown` 发出的内容）
2. `--- timeline ---`（装饰性分隔符）
3. `---` 直接在 `## Timeline` 或 `## History` 标题之前（向后兼容）

正文中的裸 `---` 现在是 markdown 水平线，而不是时间线分隔符。如果您的代理使用裸 `---` 分隔符编写页面，请迁移到 `<!-- timeline -->` —— `serializeMarkdown` 助手已经这样做了。

### 4. Wiki 子类型现在自动类型化

`inferType` 现在自动检测五个额外的目录模式作为它们自己的页面类型（以前它们都默认为 `concept`）：

| 路径模式 | 新类型 |
|------------------------|----------------|
| `/wiki/analysis/` | `analysis` |
| `/wiki/guides/` | `guide` |
| `/wiki/hardware/` | `hardware` |
| `/wiki/architecture/` | `architecture` |
| `/writing/` | `writing` |

如果您的技能或查询按 `type=concept` 过滤并期望 wiki 内容在该存储桶中，请更新它们以包含新类型。

---

## v0.13.0 — Frontmatter 关系索引

**结论：大多数技能无需操作。** v0.13 将 YAML frontmatter 字段投影到图中作为类型化边。摄入 API 不变 —— 继续按今天的方式使用 frontmatter 调用 `put_page`；图在幕后自动填充。

如果要使用新的 `auto_links.unresolved` 响应字段，三个技能会获得一个可选的新阶段。否则，无法解析的 frontmatter 名称会静默跳过（与 v0.12 行为相同）。

### 1. meeting-ingestion/SKILL.md（可选）

**位置：** 在 "Phase 3: Write Meeting Page" 之后添加新部分。

```markdown
### Phase 3.5: Check for unresolved attendees (v0.13+)

After `put_page`, inspect `response.auto_links.unresolved` — an array of frontmatter
references that did not resolve to existing pages. For meetings, this usually means
attendees you haven't created a person page for yet.

If `unresolved.length > 0`:
- Option 1 (create pages now): trigger an enrichment pass to build the missing people pages.
- Option 2 (defer): log the unresolved names to the enrichment queue for later.
- Option 3 (accept the gap): the attendee edge will not be created until a page exists.
  Re-running `gbrain extract links --source db --include-frontmatter` after creating
  the page fills in the missing edges.
```

### 2. enrich/SKILL.md（可选）

**位置：** 添加到丰富触发列表。

```markdown
### Drain unresolved frontmatter names (v0.13+)

If any `put_page` response includes `auto_links.unresolved` entries, the enrichment
tier should pick up those (field, name) pairs and try to create the missing entity
pages. Example flow:

1. signal-detector captures a meeting with `attendees: [Alice Known, Unknown Person]`
2. put_page returns `auto_links.unresolved = [{field: 'attendees', name: 'Unknown Person'}]`
3. enrichment tier consumes `Unknown Person` → web search → creates `people/unknown-person.md`
4. The next put_page (or a backfill run) wires up the `attended` edge automatically
```

### 3. idea-ingest/SKILL.md（可选）

**位置：** 与 meeting-ingestion 相同的模式 —— 在 `put_page` 后检查 `auto_links.unresolved`，将名称路由到丰富。

### 未更改的技能（无需差异）

- **brain-ops/SKILL.md** — auto-link 机制是内部的；写路径保持不变。
- **signal-detector/SKILL.md** — 信号捕获路径不变。
- **query/SKILL.md** — `traverse_graph` 现在自动返回更丰富的结果。
- **daily-task-manager/SKILL.md**、**briefing/SKILL.md**、**citation-fixer/SKILL.md**、**media-ingest/SKILL.md** — 不变。

### 可在图查询中过滤的新边类型

v0.13 边带有新的 `link_type` 值。如果您的 fork 有按类型过滤的图查询技能，现在可以使用这些：

- `works_at`（person → company）—— 来自 `company:`、`companies:` 或 `key_people:`
- `founded`（person → company）—— 来自 `founded:`
- `invested_in`（investor → deal/company）—— 来自 `investors:` 或 `lead:`
- `led_round`（lead → deal）—— 来自 `lead:`
- `yc_partner`（partner → company）—— 来自 `partner:`
- `attended`（person → meeting）—— 来自 `attendees:`
- `discussed_in`（source → page）—— 来自 `sources:`
- `source`（page → source）—— 来自 `source:`
- `related_to`（page → target）—— 来自 `related:` 或 `see_also:`

### 迁移时间

`gbrain upgrade` 在 46K 页面的大脑上需要 2-5 分钟（一次性）。通过 `gbrain post-upgrade` 在进程外运行。如果您的代理在升级期间持有数据库连接，请在之后重新连接；否则继续服务。

### v0.13 中不包含类型规范化

具有 `link_type='attendee'` 或 `link_type='mention'` 的遗留行与新的 `'attended'` / `'mentions'` 行共存。您对旧类型名称过滤的查询继续工作。v0.14 中的单独 opt-in `gbrain normalize-types` 命令处理重命名。

## v0.14.0 shell jobs（可选采用，无需技能编辑）

向 Minions 添加 `shell` 作业类型，以便确定性 cron 脚本（API 获取、令牌刷新、抓取 + 写入）从 LLM 网关移出。每次触发零令牌。在典型规模下约 60% 的网关 CPU 余量。功能**默认关闭**，现有安装继续完全按以前的方式运行。没有任何东西会中断。

要采用，请遵循 `skills/migrations/v0.14.0.md`。简短版本：

1. 在 worker 进程上设置 `GBRAIN_ALLOW_SHELL_JOBS=1`，然后 `gbrain jobs work`（Postgres）。在 PGLite 上，每个 crontab 调用使用 `--follow` 进行内联执行；没有持久 worker。
2. 对主机的每个 cron 条目进行分类：需要 LLM（保留在网关上）vs 确定性（shell 候选）。典型分类：
   - **确定性 → shell：** `ycli-token-refresh`、`x-oauth2-refresh`、`x-garrytan-unified`、`calendar-sync-to-brain`、`github-pulse`、`frameio-scan`、`flight-tracker`、`x-raw-json-backfill`。
   - **需要 LLM → 保留：** `social-radar`、`content-ideas`、`adversary-vacuum`、`ea-inbox-sweep`、`morning-briefing`、`brain-maintenance`。
3. 对于每个确定性 cron，重写为：
   ```cron
   3 13,16,19,22,1,4,7,10 * * * \
     gbrain jobs submit shell \
       --params '{"cmd":"node scripts/your-script.mjs","cwd":"/data/.openclaw/workspace"}' \
       --max-attempts 3 --timeout-ms 300000
   ```
4. 观察 `gbrain jobs get <id>` 获取每次触发的 exit_code / stdout_tail / stderr_tail。在批准下一批之前，与迁移前的行为进行比较。

**无需技能编辑。** 处理程序在 worker 端运行；技能文件不变。如果您的主机通过插件契约（v0.11.0）公开自定义处理程序，它们仍然以相同的方式工作。

铁律：**永远不要自动重写操作员的 crontab。** 每次重写都是每个 cron、人工批准的，并带有差异。如果您以后想要自动化，即将推出的 `gbrain crontab-to-minions <file>` 助手是 TODOS 中的 P1。

---

## v0.16.0: durable agent runtime

v0.15 发布了 `gbrain agent run` / `gbrain agent logs`、Minions 中新的 `subagent` 处理程序类型，以及主机仓库子代理定义的插件契约。现有技能都不需要修改。下游代理的问题是*如何*采用新运行时，而不是如何修补破坏性更改。

### 1. 使用 Anthropic 密钥运行 worker

子代理处理程序（`subagent` 和 `subagent_aggregator`）始终在 worker 上注册。没有单独的 opt-in 标志 —— `ANTHROPIC_API_KEY` 是自然的成本门（没有密钥，SDK 调用在第一轮失败），并且谁可以提交已经受到保护（`PROTECTED_JOB_NAMES` + trusted-submit：MCP 调用者获得 `permission_denied`；只有 `gbrain agent run` 可以插入这些行）。

```bash
ANTHROPIC_API_KEY=sk-ant-... gbrain jobs work
```

Worker 启动时打印：

```
[minion worker] subagent handlers enabled
```

### 2. 将子代理作为插件发布（OpenClaw + 类似）

将自定义子代理定义从 gbrain fork 移出，并作为插件放入您自己的仓库。具体来说：

```
~/<your-agent>/gbrain-plugin/
├── gbrain.plugin.json
└── subagents/
    ├── meeting-ingestion.md
    ├── signal-detector.md
    └── daily-task-prep.md
```

`gbrain.plugin.json`：

```json
{
  "name": "your-openclaw",
  "version": "2026.4.20",
  "plugin_version": "gbrain-plugin-v1"
}
```

每个 `subagents/*.md` 都是纯文本代理定义 —— YAML frontmatter + body-as-system-prompt。识别的 frontmatter 字段：`name`、`model`、`max_turns`、`allowed_tools`（必须是派生的 brain-tool 注册表的子集）。

启用它：

```bash
export GBRAIN_PLUGIN_PATH="$HOME/<your-agent>/gbrain-plugin"
```

Worker 启动时为每个插件打印 `[plugin-loader] loaded '<name>' v<ver> (N subagents)`；任何拒绝（错误的清单、`allowed_tools` 中的未知工具、版本不匹配）都会在启动时显示为明显警告，而不是静默的调度时失败。有关完整契约，请参见 `docs/guides/plugin-authors.md`。

### 3. 用持久的子代理替换临时子代理运行

如果您的代理当前生成临时子代理（OpenClaw `Agent()`、临时 Anthropic API 调用等）来处理应该在崩溃、睡眠或 worker 重启后存活的工作，请将这些迁移到 `gbrain agent run`。持久性是免费的：

```bash
gbrain agent run "analyze my last 50 journal pages for recurring themes" \
  --subagent-def analyzer --fanout-manifest manifests/journal-pages.json
```

每个回合都持久化到 `subagent_messages`，每个工具调用都是两阶段分类账，并且 `gbrain agent logs <job>` 显示它在哪里失败 + 最后一次成功调用返回了什么。不再有"因为会话上下文消失而从头重新运行"。

### 4. 来自子代理的 `put_page` 在代理命名空间下写入

如果您采用了 v0.15 子代理运行时，请注意源自子代理工具调度的 `put_page` 调用**必须**针对 `wiki/agents/<subagent_id>/...`。显示给模型的架构在第一次尝试时强制执行此操作；服务器端的 fail-closed 检查拒绝其他任何内容。这**不**影响您的技能文件、CLI put_page 调用或 MCP put_page —— 只影响 LLM 循环内的工具调度写入。

聚合输出（最终的"这是所有 N 个子项发现的内容"大脑页面）通过单独的可信 CLI 路径进行，而不是通过子代理工具调用，因此它可以写入您想要的任何位置。

铁律：**永远不要授予代理超出其命名空间的写入权限。** 服务器端检查存在是因为调度程序错误会发生；将其视为深度防御，而不是主要边界。

---

## v0.22.4 — frontmatter-guard 采用

### 1. 停止手工编写 frontmatter 验证器

如果您的 fork 有直接调用 `js-yaml` 来验证大脑页面 frontmatter 的脚本，请将它们替换为 `gbrain frontmatter validate` 调用。CLI 涵盖七个规范错误类，并提供跨版本稳定的 `--json` 包络。

```diff
- # Custom validator script
- node scripts/validate-frontmatter.mjs <path>
+ gbrain frontmatter validate <path> --json
```

对于需要在另一个脚本中使用验证器的使用者，从 gbrain 的 `markdown` 导出导入，而不是重复逻辑：

```ts
import { parseMarkdown } from 'gbrain/markdown';

const parsed = parseMarkdown(content, filePath, { validate: true, expectedSlug });
for (const err of parsed.errors ?? []) {
  // err.code: MISSING_OPEN | MISSING_CLOSE | YAML_PARSE | SLUG_MISMATCH |
  //           NULL_BYTES | NESTED_QUOTES | EMPTY_FRONTMATTER
}
```

### 2. 删除对 `lib/brain-writer.mjs` 的任何引用

如果您的 fork 的技能或脚本引用了一个有抱负的 `lib/brain-writer.mjs`（它从未发布 —— 规范在 PR #392 中，从未落地），请将这些引用替换为 gbrain CLI。`frontmatter-guard` 技能位于 `skills/frontmatter-guard/SKILL.md`，指向 `gbrain frontmatter validate` / `audit` / `install-hook`。

### 3. 将 doctor 子检查连接到您的健康管道

`gbrain doctor` 现在自动报告 `frontmatter_integrity`。如果您的 fork 有自定义健康管道（例如关于大脑健康的每日 Slack 帖子），从 `gbrain doctor --json` 提取并显示 `frontmatter_integrity` 行数。

### 4.（可选）在大脑仓库上安装 pre-commit hook

对于由 git 支持的源，v0.22.4 install-hook 助手会放置一个 pre-commit 脚本，阻止带有格式错误 frontmatter 的提交：

```bash
gbrain frontmatter install-hook
```

如果您的大脑不是 git 仓库，或者您的下游代理已经在写入时强制执行验证，请跳过此步骤。有关完整方案，请参见 `docs/integrations/pre-commit.md`。

### 5. 迁移人体工程学 — 读取 pending-host-work.jsonl

`gbrain apply-migrations --yes` 运行 v0.22.4 审计后，您的代理应该读取 `~/.gbrain/migrations/pending-host-work.jsonl`（过滤到 `migration === "0.22.4"`）并遍历每个条目的 `command` 字段。每个条目指向每个源的 `gbrain frontmatter validate <source_path> --fix` 命令 —— 向用户显示计数，获得明确同意，然后运行。

迁移是**仅审计**。它在 `apply-migrations` 期间从不改变大脑内容。您的代理在用户同意的情况下运行修复命令。

---

## 未来版本

当 gbrain 发布新版本时，本文档将更新该版本的差异。每个新版本追加一个部分；旧部分保留，以便您可以一次赶上多个版本。

要检查您的 fork 缺少什么：
```bash
diff <(grep -A3 "Based on gbrain" ~/<your-fork>/skills/brain-ops/SKILL.md) \
     <(grep "v[0-9]" ~/gbrain/skills/migrations/ | tail -3)
```