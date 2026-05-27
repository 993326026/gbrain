# GBrain 安装验证手册

安装后运行这些检查以确认 GBrain 的每个部分都正常工作。
每个检查包括命令、预期输出以及失败时的处理方法。

最重要的检查是 #4（实时同步）。"同步已运行"不等于"同步正常工作"。由于池器错误而静默跳过页面的同步比不同步更糟，因为你认为它正在工作。

---

## 1. Schema 验证

**命令：**

```bash
gbrain doctor --json
```

**预期结果：** 所有检查返回 `"ok"`：
- `connection`: 已连接，N 个页面
- `pgvector`: 扩展已安装
- `rls`: 所有表上已启用
- `schema_version`: 当前版本
- `embeddings`: 覆盖率百分比

**如果失败：** doctor 输出包含每个检查的具体修复说明。参见 `skills/setup/SKILL.md` 错误恢复表。

---

## 2. 技能包已加载

**检查：** 向代理询问："What is the brain-agent loop?"

**预期结果：** 代理引用 GBRAIN_SKILLPACK.md 第 2 节并描述读写循环：检测实体、读取大脑、用上下文响应、写入大脑、同步。

**如果失败：** 代理未加载技能包。运行安装粘贴中的步骤 6（阅读 `docs/GBRAIN_SKILLPACK.md`）。

---

## 3. 自动更新已配置

**命令：**

```bash
gbrain check-update --json
```

**预期结果：** 返回包含 `current_version`、`latest_version`、`update_available`（布尔值）的 JSON。已注册 cron `gbrain-update-check`。

**如果失败：** 运行安装粘贴中的步骤 7。参见 GBRAIN_SKILLPACK.md 第 17 节。

---

## 4. 实时同步确实有效

这是最重要的检查。分为三部分。

### 4a. 覆盖率检查

比较数据库中的页面计数与仓库中的可同步文件计数：

```bash
gbrain stats
```

然后计算可同步文件：

```bash
find /data/brain -name '*.md' \
  -not -path '*/.*' \
  -not -path '*/.raw/*' \
  -not -path '*/ops/*' \
  -not -name 'README.md' \
  -not -name 'index.md' \
  -not -name 'schema.md' \
  -not -name 'log.md' \
  | wc -l
```

**预期结果：** `gbrain stats` 中的页面计数应接近文件计数。有些差异是正常的（自上次同步以来添加的文件），但如果页面计数小于文件计数的一半，同步正在静默跳过页面。

**如果页面计数太低：** 首要原因是连接池器错误。检查你的 `DATABASE_URL`：
- 如果包含 `pooler.supabase.com:6543`，验证它使用的是 **Session 模式**，而不是 Transaction 模式。
- Transaction 模式会破坏 `engine.transaction()` 并导致 `.begin() is not a function` 错误。
- 修复：切换到 Session 模式池器字符串，然后运行 `gbrain sync --full` 重新导入所有内容。

### 4b. 嵌入检查

```bash
gbrain stats
```

**预期结果：** 嵌入块计数应接近总块计数。

**如果嵌入远低于总数：**

```bash
gbrain embed --stale
```

如果未设置 `OPENAI_API_KEY`，则无法生成嵌入。没有嵌入，关键词搜索仍然有效，但混合/语义搜索将无效。

### 4c. 端到端测试

这是真正的测试。编辑大脑页面，推送，等待，搜索。

1. 在大脑仓库中编辑一个页面（例如，更正某人页面上的事实）：

```bash
# 示例：修复 Gustaf 页面中的一行
cd /data/brain
# 对任何 .md 文件进行小修改
git add -A && git commit -m "test: verify live sync" && git push
```

2. 等待下一个同步周期（cron 间隔或 `--watch` 轮询）。

3. 搜索更正后的文本：

```bash
gbrain search "<text from the correction>"
```

**预期结果：** 搜索返回 **更正后的** 文本，而不是旧版本。

**如果返回旧文本：** 同步静默失败。检查：
- 同步 cron 是否已注册并运行？
- `gbrain sync --watch` 是否仍然运行（如果使用 watch 模式）？
- 运行 `gbrain config get sync.last_run` 查看同步上次运行时间。
- 手动运行 `gbrain sync --repo /data/brain` 并检查错误。
- 如果看到 `.begin() is not a function`，修复池器（参见上面的 4a）。

---

## 5. 嵌入覆盖率

**命令：**

```bash
gbrain stats
```

**预期结果：** 嵌入块计数匹配（或接近）总块计数。

**如果为零或非常低：** `OPENAI_API_KEY` 可能缺失或无效。检查：

```bash
echo $OPENAI_API_KEY | head -c 10
```

如果为空，设置密钥。然后：

```bash
gbrain embed --stale
```

---

## 6. 大脑优先查找协议

**检查：** 向代理询问大脑中存在的人或概念。

**预期结果：** 代理首先使用 `gbrain search` 或 `gbrain query`，而不是 grep 或外部 API。响应包含来自大脑的上下文和来源归属。

**如果失败：** 大脑优先查找协议未注入到代理的系统上下文中。参见 `skills/setup/SKILL.md` Phase D。

---

## 7. 知识图谱已连接

v0.12.0 图层需要为现有大脑填充。新写入会自动链接，但历史页面需要一次性回填。

**命令：**

```bash
gbrain stats | grep -E 'links|timeline'
```

**预期结果：** `links` 和 `timeline_entries` 都非零（假设大脑包含带有实体引用和日期标记的内容）。

**如果在有导入内容的大脑上为零：** 运行回填。

```bash
gbrain extract links --source db --dry-run | head -5    # 预览
gbrain extract links --source db                         # 提交
gbrain extract timeline --source db
gbrain stats                                             # 确认 > 0
```

**额外检查** — 图遍历工作：

```bash
# 从你的大脑中选择任何连接良好的 slug
gbrain graph-query people/<some-person-slug> --depth 2
```

**预期结果：** 类型化边的缩进树（`--attended-->`, `--works_at-->`, 等）。如果 slug 没有入站或出站链接，尝试其他 slug 或再次运行提取。

**如果提取找不到任何内容：** 你的页面可能未使用实体引用语法。提取器匹配 `[Name](people/slug)`、`[Name](../people/slug.md)` 和裸 `people/slug` 引用。如果你的大脑使用不同的格式，自动链接启发式将找不到它们 —— 使用示例页面提交 issue。

---

## 8. JSONB Frontmatter 完整性（v0.12.2）

v0.12.2 之前创建的 Postgres 大脑具有双重编码的 JSONB 列（`frontmatter->>'key'` 返回 NULL，GIN 索引无效）。`gbrain upgrade` 通过 `v0_12_2` 编排器自动运行 `gbrain repair-jsonb`。验证修复是否成功。

**命令：**

```bash
gbrain repair-jsonb --dry-run --json
```

**预期结果：** 所有 5 列（`pages.frontmatter`、`raw_data.data`、`ingest_log.pages_updated`、`files.metadata`、`page_versions.frontmatter`）的 `totalRepaired: 0`。零计数意味着每一行都是正确类型的 JSON 对象，而不是字符串编码的 JSON。

**如果计数 > 0：** 修复未运行或被中断。不带 `--dry-run` 重新运行：

```bash
gbrain repair-jsonb
```

幂等操作。PGLite 大脑始终报告 0（不受原始错误影响）。

**额外检查** — frontmatter 键查询实际解析：

```bash
gbrain call list_pages '{"frontmatterKey": "type", "frontmatterValue": "person"}'
```

如果在有人物页面的大脑上返回行，则 JSONB 路径是健康的。

---

## 快速验证（一次运行所有检查）

```bash
# 1. Schema
gbrain doctor --json

# 2. 同步最近性
gbrain config get sync.last_run

# 3. 页面计数 + 嵌入覆盖率
gbrain stats

# 4. 搜索工作
gbrain search "test query from your brain content"

# 5. 捕获任何未嵌入的块
gbrain embed --stale

# 6. 自动更新
gbrain check-update --json

# 7. 知识图谱已填充（links + timeline > 0）
gbrain stats | grep -E 'links|timeline'

# 8. JSONB 完整性（v0.12.2 — 仅 Postgres，PGLite 始终为 0）
gbrain repair-jsonb --dry-run --json
```

如果所有八项都成功返回，则安装是健康的。对于完整的端到端同步测试（4c），推送真正的更改并验证它出现在搜索中。