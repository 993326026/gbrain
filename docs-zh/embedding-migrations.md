# 在现有大脑上切换嵌入模型或维度

GBrain 在 `content_chunks` 的固定维度 `vector(N)` 列中存储嵌入。如果您切换到不同维度的模型（例如 `text-embedding-3-large` 1536 → `voyage-multilingual-large-2` 2048，或切换回较小的模型如 `nomic-embed-text` 768），磁盘上的列类型不会自动更改。

`gbrain init` 和 `gbrain doctor` 都会检测到这种情况并拒绝静默继续。本文档是它们指向的方案。

## 为什么我们不自动执行此操作

切换维度需要：

1. 删除 HNSW 向量索引（pgvector 无法在 `ALTER COLUMN TYPE` 后存活）。
2. 更改列类型。
3. 擦除所有现有嵌入（旧向量在新空间中无法使用）。
4. 重新嵌入整个语料库（在 50K 页面的大脑上可能需要数小时，根据模型的不同，API 调用成本为 $1-100）。
5. 有条件地重新创建索引（pgvector 的 HNSW 支持最多 2000 维；超过该值必须使用精确扫描）。

这不是升级时的自动运行。这是一个深思熟虑、代价高昂的操作。当您确定要使用新模型时再运行它。

## 方案 — 针对您的大脑手动执行 `psql`

将 `<NEW_DIMS>` 替换为您的目标维度数。

```sql
BEGIN;

-- 1. 删除 HNSW 索引。它无法在列类型更改后存活。
DROP INDEX IF EXISTS idx_chunks_embedding;

-- 2. 更改列类型。（如果现有数据已经消失，您可以使用 DROP COLUMN + ADD COLUMN 代替 —— 最终状态相同。）
ALTER TABLE content_chunks ALTER COLUMN embedding TYPE vector(<NEW_DIMS>);

-- 3. 清除过期的嵌入，使其不会存活到新空间中。
--    可以截断（更快，删除所有块）或置空（保留块文本，以便重新嵌入时无需重新分块）：
UPDATE content_chunks SET embedding = NULL, embedded_at = NULL;

-- 4. 仅在 dims <= 2000 时重新创建 HNSW 索引。超过该值时，保留无索引状态并依赖精确扫描（gbrain searchVector 会自动处理这一点 —— 搜索只会变慢，不会中断）。
-- 对于 dims <= 2000（例如 1024, 1536, 768）：
CREATE INDEX IF NOT EXISTS idx_chunks_embedding
  ON content_chunks USING hnsw (embedding vector_cosine_ops);
-- 对于 dims > 2000（例如 2048 Voyage 4 Large）：跳过步骤 4。

COMMIT;
```

然后更新 gbrain 的配置，使其知道新的维度：

```bash
gbrain config set embedding_model <model>
gbrain config set embedding_dimensions <NEW_DIMS>
```

并重新嵌入语料库：

```bash
gbrain embed --stale
```

## PGLite（本地大脑）

相同的方案，但连接到嵌入式数据库的方式不同：

```bash
gbrain config get database_url   # confirm engine: pglite
# 打开 psql 等效工具 —— 对于 PGLite，最简单的方法是编写一个小脚本，导入 PGLiteEngine 并通过 engine.executeRaw 运行 SQL。
# 或者临时迁移到 Postgres（gbrain migrate --to supabase），如果您想要真正的 psql 连接。
```

对于大多数 PGLite 用户，如果您的语料库足够小，重新同步比手动编写迁移更快，那么更简单的方法是**擦除并重新初始化**：

```bash
mv ~/.gbrain/brain.pglite ~/.gbrain/brain.pglite.bak
gbrain init --pglite --embedding-dimensions <NEW_DIMS>
gbrain sync   # 从磁盘重新导入您的大脑仓库
```

## 验证

方案实施后，`gbrain doctor --fast` 应该报告正常，`gbrain doctor`（完整）应该说检查 8b 通过：

```
✓ embedding_provider     dim parity: config 768 / column vector(768) / live probe 768
```

如果没有，请提交 issue，附上 doctor 输出和您运行的 SQL。

## v0.29+ 计划

`gbrain migrate-embedding-dim --to <N>` 是一个已跟踪的 TODO。它将运行上述方案，带有进度报告 + 明确的确认门。在此之前，此手动方案是规范路径。