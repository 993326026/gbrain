# 存储分层：db-tracked vs db-only 目录

## 概述

GBrain 支持存储分层，将版本控制内容与批量机器生成数据分离。这可防止 git 仓库因大量自动生成的内容而膨胀，同时仍将其保留在数据库中。

> 命名注意：在 v0.22.11 之前，键是 `git_tracked` / `supabase_only`。现在的标准名称是 `db_tracked` / `db_only`（引擎无关 —— 在 PGLite 和 Postgres 上都有效）。已弃用的键仍然加载，但每个进程会警告一次。当该路径落地时，运行 `gbrain doctor --fix` 进行自动重命名。

## 配置

在大脑仓库根目录的 `gbrain.yml` 文件中添加 `storage` 部分：

```yaml
storage:
  # 版本控制的目录（人工编辑，提交到 git）。
  db_tracked:
    - people/
    - companies/
    - deals/
    - concepts/
    - yc/
    - ideas/
    - projects/

  # 仅通过大脑数据库持久化的目录（批量机器生成的内容）。作为本地缓存写入磁盘但不提交到 git；`gbrain sync` 自动管理这些路径的 .gitignore。`gbrain export --restore-only` 从数据库重新填充缺失的文件。
  db_only:
    - media/x/
    - media/articles/
    - meetings/transcripts/
```

路径要求：

- 每个目录必须以 `/` 结尾（标准形式）。验证器自动规范化缺少的尾部斜杠（一次性信息注释显示更改内容）。
- 一个目录不能同时出现在两个层中 —— 这是层重叠错误，`loadStorageConfig` 抛出 `StorageConfigError`。编辑 `gbrain.yml` 以删除重叠并重试。

## 行为变更

### 1. `gbrain sync` — 自动 .gitignore 管理

当存在存储配置时，`gbrain sync` 在每次成功同步时自动管理 `.gitignore` 条目：

- 将缺失的 `db_only` 目录模式添加到 `.gitignore`。
- 幂等 —— 重新运行不会添加重复条目。
- 稳定的注释头，因此管理块可被 grep。
- 在 `--dry-run` 上跳过（预览模式下不修改磁盘）。
- 在 `blocked_by_failures` 状态下跳过（同步状态不一致）。
- 当仓库是 git submodule 时跳过（`.git` 是文件，不是目录）—— submodule 的 .gitignore 更改在父更新后不会保留。显示警告说明。
- 设置 `GBRAIN_NO_GITIGNORE=1` 时完全跳过（共享仓库设置的逃生舱口，维护者希望 gbrain 不修改 .gitignore）。
- 失败（写入权限被拒绝等）被捕获并记录，永不崩溃同步。

示例 `.gitignore` 添加：

```gitignore
# Auto-managed by gbrain (db_only directories)
media/x/
media/articles/
meetings/transcripts/
```

### 2. `gbrain export --restore-only` — 重新填充缺失的 db_only 文件

```bash
# 仅从数据库恢复缺失的 db_only 文件。
gbrain export --restore-only --repo /path/to/brain

# 按页面类型过滤。
gbrain export --restore-only --type media --repo /path/to/brain

# 按 slug 前缀过滤。
gbrain export --restore-only --slug-prefix media/x/ --repo /path/to/brain

# 组合过滤器。
gbrain export --restore-only --type media --slug-prefix media/x/ --repo /path/to/brain
```

`--restore-only` 标志：

- 通过链 `--repo` → 类型化 `sources.getDefault()` → 硬错误解析 repoPath。从不回退到当前目录。
- 仅导出匹配 `db_only` 模式且磁盘上缺失的页面。
- 非常适合容器重启恢复和新克隆。

### 3. `gbrain storage status` — 存储层健康仪表板

```bash
# 人类可读状态。
gbrain storage status --repo /path/to/brain

# JSON 输出用于脚本和编排器。
gbrain storage status --repo /path/to/brain --json
```

输出包括：

- 按存储层的总页面计数。
- 按层的磁盘使用细分。
- 需要恢复的缺失文件（显示前 10 个；完整列表在 `--json` 中）。
- 配置验证警告。
- 当前层目录列表。

示例输出：

```
Storage Status
==============

Repository: /data/brain
Total pages: 15,243

Storage Tiers:
-------------
DB tracked:     2,156 pages
DB only:        12,887 pages
Unspecified:    200 pages

Disk Usage:
-----------
DB tracked:     45.2 MB
DB only:        2.1 GB

Missing Files (need restore):
-----------------------------
  media/x/tweet-1234567890
  media/x/tweet-0987654321
  ... and 47 more

Use: gbrain export --restore-only --repo "/data/brain"

Configuration:
--------------
DB tracked directories:
  - people/
  - companies/
  - deals/

DB-only directories:
  - media/x/
  - media/articles/
  - meetings/transcripts/
```

## 验证

`loadStorageConfig` 在解析后运行 `normalizeAndValidateStorageConfig`：

- 自动修复（静默，带一次性信息注释显示更改内容）：
  - 添加缺失的尾部 `/`：`'media/x'` → `'media/x/'`。
- 抛出 `StorageConfigError`（调用者看到干净的 exit-1 和可操作的消息）：
  - 同一目录同时在 `db_tracked` 和 `db_only` 中（路由模糊）。

## 用例

### 大脑仓库扩展

非常适合超过 50K-200K+ 文件的大脑仓库：

- 核心知识（人物、公司、交易）保持 git 跟踪。
- 批量数据（推文、文章、转录）移至 db_only。
- 较小的 git 仓库保持开发速度。
- 通过数据库仍可访问完整数据。

### 基于容器的部署

对于临时容器环境必不可少：

- Git 仓库仅包含必要文件。
- 容器重启不会丢失 db_only 数据。
- `gbrain export --restore-only` 在需要时快速恢复批量文件。
- 本地磁盘充当缓存层。

### 多环境一致性

支持跨环境的一致数据访问：

- 开发：小型 git 克隆，按需恢复批量数据。
- 生产：通过数据库访问完整数据集，选择性本地缓存。
- CI/CD：仅使用 git 跟踪数据进行快速测试。

## 迁移策略

1. **评估当前仓库**：使用 `gbrain storage status` 了解当前分布。
2. **规划目录结构**：确定哪些目录应该是 db_tracked vs db_only。
3. **创建 `gbrain.yml`**：在仓库根目录添加存储配置。
4. **使用 dry-run 测试**：`gbrain sync --dry-run` 验证行为；dry-run 不修改 `.gitignore`。
5. **运行真正的同步**：`gbrain sync` 在成功时自动更新 `.gitignore`。
6. **验证恢复**：针对小型 db_only 目录测试 `gbrain export --restore-only --repo .`。

## 最佳实践

- **目录命名**：存储路径以 `/` 结尾（标准形式）。如果你忘记，验证器会规范化。
- **从小处着手**：从 `db_only` 中明确的机器生成目录开始。
- **处理验证错误**：层重叠是错误，不是警告。在同步前修复它。
- **测试恢复**：定期在 staging 环境中测试 `--restore-only`。
- **记录决策**：在 `gbrain.yml` 中添加注释解释层选择。

## PGLite 引擎注意事项

在 PGLite 引擎（gbrain 的本地嵌入式 Postgres）上，你的 db_only 页面所在的"数据库"就是 gbrain 用于其他所有内容的本地文件。`.gitignore` 管理仍然有帮助（将批量内容排除在 git 历史之外），但"卸载到数据库"的承诺在技术上是空的。检测到引擎时会显示每个进程一次的软警告。要获得完整分层，请使用 `gbrain migrate --to supabase` 迁移到 Postgres。

## 兼容性

- **向后兼容**：没有 `gbrain.yml` 的系统不受影响。
- **渐进增强**：在需要时添加配置。
- **数据库不变**：所有数据无论层级都保留在 Postgres 中。
- **现有工作流**：所有现有的 `sync` 和 `export` 行为都保留。
- **已弃用的键**：`git_tracked` / `supabase_only` 仍然加载，但每个进程会警告一次。