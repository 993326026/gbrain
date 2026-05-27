# Brains and Sources — 心智模型

GBrain 有两个正交的知识组织轴。用户和代理都需要理解这两个轴，否则查询会静默地路由错误。

**TL;DR:**
- **brain** 是一个数据库。您可以有多个。
- **source** 是 brain *内部* 的命名内容仓库。一个 brain 可以包含多个。
- `--brain <id>` 选择**哪个数据库**。
- `--source <id>` 选择该数据库**内的哪个仓库**。
- 它们是独立的。您可以选择任意组合。

---

## 两个轴

### Brains（数据库轴）

**brain** 是一个数据库 —— PGLite 文件、自托管 Postgres 或 Supabase。每个 brain 有：
- 自己的 `pages` 表、`chunks` 表、`embeddings` 等。
- 如果通过 HTTP MCP 提供服务，则有自己的 OAuth 界面（v0.19+，PR 2）。
- 自己独立的生命周期、备份、访问控制。

Brains 通过以下方式枚举：
- **host** — 您的默认 brain，配置在 `~/.gbrain/config.json` 中。
- **mounts** — 通过 `gbrain mounts add <id>` 在 `~/.gbrain/mounts.json` 中注册的其他 brains（v0.19+）。

路由：`--brain <id>`、`GBRAIN_BRAIN_ID`、`.gbrain-mount` dotfile，或对已注册挂载路径的最长路径匹配。回退到 `host`。

### Sources（仓库轴，v0.18.0+）

**source** 是一个 brain *内部* 的命名内容仓库。每个 `pages` 行都带有 `source_id`。Slugs 每个 source 唯一，不是全局唯一。

示例：在一个 brain 中，slug `topics/ai` 可以存在于 `source=wiki` 下 AND `source=gstack` 下 —— 它们是不同的页面。

路由：`--source <id>`、`GBRAIN_SOURCE`、`.gbrain-source` dotfile，或 `sources` 表中已注册的 `local_path` 匹配。

### 何时移动每个轴？

| 您想要 | 调整 |
|---|---|
| 在同一个 brain 中的不同仓库工作（wiki → gstack notes） | `--source` |
| 查询不属于您的团队发布的 brain | `--brain` |
| 隔离一个主题，使其从不泄露到个人搜索中 | `--source` 配合 `federated=false` |
| 与队友共享一个 brain | `--brain`（挂载团队 brain） |
| 在您的个人 brain 中添加新仓库 | `--source` 通过 `gbrain sources add` |
| 添加团队 brain | `--brain` 通过 `gbrain mounts add` |

**经验法则：** 如果数据所有者改变，那是 brain 边界。如果数据所有者保持不变但主题/仓库改变，那是 source 边界。

---

## 拓扑：单人开发者

最简单的情况。一个 brain，一个 source。

```
┌─────────────────────────────────────────┐
│  host brain (~/.gbrain)                 │
│  ├── source: default (federated=true)   │
│  │   └── all pages                      │
└─────────────────────────────────────────┘
```

`gbrain query "retry budgets"` 找到所有内容。不需要 `--brain`，不需要 `--source`。

---

## 拓扑：具有多个仓库的个人 brain

您维护多个代码库或写作流。每个都是一个 brain 内部的独立 source。跨 source 搜索默认开启，因此关于"caching"的查询会返回来自每个仓库的结果。

```
┌──────────────────────────────────────────────┐
│  host brain (~/.gbrain)                      │
│  ├── source: wiki      (federated=true)      │
│  │   └── personal notes, people, companies   │
│  ├── source: gstack    (federated=true)      │
│  │   └── gstack plans, learnings             │
│  ├── source: openclaw  (federated=true)      │
│  │   └── openclaw docs, memos                │
│  └── source: essays    (federated=false)     │
│      └── draft essays, isolated on purpose   │
└──────────────────────────────────────────────┘
```

在 `~/openclaw/` 内部，`.gbrain-source` dotfile 将每个命令固定到 `source=openclaw`。在 `~/gstack/` 内部，dotfile 固定到 `source=gstack`。所有内容仍然指向一个数据库。

当以下情况使用此拓扑：
- 您拥有所有内容。
- 您希望跨仓库搜索正常工作。
- 您不需要与非您的人共享任何内容。

---

## 拓扑：个人 brain + 一个团队 brain

您在一个发布共享 brain 的团队中。您的个人 brain 保持原样；您将团队 brain 挂载在其旁边。

```
┌──────────────────────────────────────────────┐
│  host brain (~/.gbrain)  — YOUR personal DB  │
│  ├── source: wiki                            │
│  ├── source: gstack                          │
│  └── ...                                     │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│  mount: media-team                           │
│  path:   ~/team-brains/media                 │
│  engine: postgres (team's Supabase)          │
│  └── sources: wiki, raw, enriched            │
└──────────────────────────────────────────────┘
```

`gbrain query "X"`（无标志）→ 在 host（您的个人 brain）上运行。
`gbrain query "X" --brain media-team` → 在团队的数据库上运行。
在 `~/team-brains/media/` 内部，`.gbrain-mount` dotfile 自动将 brain 固定到 `media-team`。

当以下情况使用此拓扑：
- 您在一个团队中，有人发布了团队订阅的 brain。
- 您需要工作和个人之间的数据隔离。
- 不同的团队/组织拥有不同的 brains。

---

## 拓扑：具有多个团队成员身份的 CEO 级别用户

您足够资深，可以跨多个团队工作。您维护个人 brain（内部有 N 个 sources）AND 挂载多个工作团队 brains。每个团队 brain 本身就是 v0.18.0 意义上的多 source brain —— 内部按照团队所有者选择的方式组织。

```
┌──────────────────────────────────────────────┐
│  host brain — YOUR personal DB               │
│  ├── source: wiki                            │
│  ├── source: essays                          │
│  ├── source: gstack                          │
│  └── source: openclaw                        │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│  mount: media-team (your media team's brain) │
│  └── sources: wiki, pipeline, enriched       │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│  mount: policy-team (your policy team's)     │
│  └── sources: wiki, research, letters        │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│  mount: portfolio (another team's)           │
│  └── sources: companies, deals, diligence    │
└──────────────────────────────────────────────┘
```

在每个团队的检出目录中，`.gbrain-mount` dotfile 固定 brain。在特定子目录中，`.gbrain-source` dotfile 固定 source。因此 `cd ~/team-brains/policy/research && gbrain query "X"` 无需任何标志即可定位 `brain=policy-team, source=research`。

当以下情况使用此拓扑：
- 您跨多个团队工作。
- 每个团队拥有自己的 brain 和自己的访问策略。
- 您需要潜在空间联合（代理决定何时跨 brains 查询），而不是 SQL 联合。

在 v0.19 中，跨 brain 查询**不是确定性的**。代理看到 brain 列表并根据需要重新查询。这是一个特性 —— 它保持调试清晰和访问控制干净。

---

## 解析优先级（一页记住）

```
WHICH BRAIN (DB)?                    WHICH SOURCE (repo in DB)?
 1. --brain <id>                      1. --source <id>
 2. GBRAIN_BRAIN_ID env               2. GBRAIN_SOURCE env
 3. .gbrain-mount dotfile             3. .gbrain-source dotfile
 4. longest-prefix mount path match   4. longest-prefix source path match
 5. (reserved: brains.default v2)     5. sources.default config
 6. fallback: 'host'                  6. fallback: 'default'
```

两个轴故意遵循相同的分层模式。如果你知道一个，你就知道另一个。

---

## 面向阅读本文档的代理

- 用户提问时的默认假设：从当前 brain 开始（通过上面的优先级解析）。不要无故切换 brains。
- 如果用户提出的问题跨越团队可能拥有的主题领域（例如"上周 X 团队决定了什么？"），正确的做法是*显式查询团队的 brain*，而不是用"team x"搜索 host。
- 跨 brain 联合是**您的工作**，不是数据库的。您有 brain 列表（`gbrain mounts list`）。您决定何时展开。您综合发现。您引用 `brain:source:slug`。
- 写入页面时，尊重 brain 边界。关于团队工作的事实属于团队的 brain，而不是用户的个人 brain。跨 brain 写入前请询问。
- 有关完整决策表，请参阅 `skills/conventions/brain-routing.md`。

## 面向阅读本文档的用户

- **默认路径：** 设置您的个人 brain（`gbrain init`），为您关心的每个仓库添加一个 source（`gbrain sources add gstack --path ~/gstack`）。您几乎永远不需要 `--brain`。
- **当团队发布 brain 时：** `gbrain mounts add <team-id> --path <clone> --db-url <url>`，该检出目录中的 `.gbrain-mount` dotfile 自动将查询路由到那里。
- **当您是具有多个团队成员身份的 CEO 级别用户时：** 挂载每个团队 brain。信任解析器 —— 在团队目录内，dotfile 选择 brain；在子目录内，dotfile 选择 source。标志用于您想要故意跨边界查询时。

## 进一步阅读

- v0.18.0 CHANGELOG — 引入了 `sources` 原语。
- v0.19.0 CHANGELOG（PR 0+1+2 发布后待定）—— 引入 `mounts`。
- `docs/mounts/publishing-a-team-brain.md` (PR 2) — 如何成为 brain 发布者，而不仅仅是订阅者。