# RLS 与你

简短版：gbrain `public` 模式中的每个表都需要启用行级安全性（RLS）。如果有任何表未启用，`gbrain doctor` 现在会失败（不是警告），并且进程以退出码 1 退出。

本指南解释原因、遇到检查时该做什么，以及确实希望表保持对 anon key 可读的情况下的逃生舱口。

## RLS 为什么重要

Supabase 通过 PostgREST 暴露 `public` 模式中的所有内容。那里的任何东西都可以被 anon key 访问，anon key 本质上是客户端机密。如果 public 表上的 RLS 关闭，anon key 可以读取它。对于任何敏感数据（auth tokens、聊天历史、财务数据），这是一个数据泄露向量，而不是小问题。

gbrain 的 service-role 连接拥有 `BYPASSRLS`，因此启用 RLS 而不添加策略**不会**破坏 gbrain 本身。它只是阻止 anon key 的默认读取。这就是安全态势：对 anon 默认拒绝，服务角色完全访问。

## doctor 失败时该做什么

Doctor 的消息会列出每个缺少 RLS 的表，并为每个表提供一条 `ALTER TABLE` 命令：

```
1 table(s) WITHOUT Row Level Security: expenses_ramp.
Fix: ALTER TABLE "public"."expenses_ramp" ENABLE ROW LEVEL SECURITY;
If a table should stay readable by the anon key on purpose, see
docs/guides/rls-and-you.md for the GBRAIN:RLS_EXEMPT comment escape hatch.
```

99% 的情况下，你需要这个修复。运行 SQL。重新运行 `gbrain doctor`。完成。

## v0.26.7 — 自动 RLS 事件触发器和一次性回填

从 v0.26.7（迁移 v35）开始，gbrain 发布了两项更改，关闭了表在 `public` 模式中可能存在任何时间而没有 RLS 的漏洞。

**1. 事件触发器。** 名为 `auto_rls_on_create_table` 的 Postgres DDL 事件触发器会对每个新创建的 `public.*` 表运行 `ALTER TABLE … ENABLE ROW LEVEL SECURITY`。它覆盖 `CREATE TABLE`、`CREATE TABLE AS … SELECT` 和 `SELECT … INTO` — Postgres 报告为表创建命令的每种语法。gbrain 本身创建的表、共享同一 Supabase 项目的其他应用（Baku、Hermes、任何东西）创建的表，或人类运行原始 SQL 创建的表，在它们存在的那一刻都会启用 RLS。非 `public` 模式（`auth`、`storage`、`realtime` 等）被显式忽略 — Supabase 管理这些，我们不应该触碰它们。

**2. 一次性回填。** 升级到 v0.26.7 时，迁移会遍历所有现有的 `public.*` 基表，这些表的 RLS 关闭且注释不包含 `GBRAIN:RLS_EXEMPT` 豁免（见下文），并对每个表启用 RLS。升级后，`gbrain doctor` 的 `rls` 检查在每个大脑上都应该是无操作。

### 破坏性变更：升级前阅读

如果你有故意关闭 RLS 并希望保持这种状态的公共表，你必须在运行 `gbrain upgrade` 到 v0.26.7 **之前**添加 `GBRAIN:RLS_EXEMPT` 注释。回填会对任何不携带下文记录的精确注释契约的公共表启用 RLS。迁移没有 `--dry-run` 标志。

弄错的最低成本是一次往返：操作员运行 SQL 在应该豁免的表上启用 RLS，然后运行 `ALTER TABLE … DISABLE ROW LEVEL SECURITY` 并添加豁免注释以防止在后续 doctor 运行时重新启用。不会丢失数据。

### 跨应用影响

如果非 gbrain 应用（Baku、Hermes、你写的脚本、任何东西）在同一个 Supabase 项目中创建表，触发器也会在这些表上启用 RLS。有两种处理方式：

1. **应用的连接角色拥有 BYPASSRLS**（例如它也使用 `postgres` 角色）。新创建的表启用 RLS，但应用可以自由读写，因为 BYPASSRLS 完全绕过策略。
2. **应用的角色没有 BYPASSRLS**。那么应用需要在创建表后立即添加 `CREATE POLICY`，授予自己所需的读写访问权限。触发器**不会**添加策略 — 它只启用 RLS，在应用的策略落地之前保持默认拒绝状态。

如果两个条件都不成立，应用将无法读取自己新创建的表。修复在应用端，而不是 gbrain：要么授予 BYPASSRLS，要么发布策略。

### 如果触发器被删除怎么办？

`gbrain doctor` 包含一个新的 `rls_event_trigger` 检查，验证触发器是否安装并启用。如果你出于任何原因手动删除它（调试、迁移测试、任何东西），doctor 会警告并给出恢复命令：

```
gbrain apply-migrations --force-retry 35
```

重新运行迁移 v35 是幂等的 — 它 `DROP EVENT TRIGGER IF EXISTS` 并干净地重新创建。

### 为什么不使用 FORCE ROW LEVEL SECURITY？

Postgres 有两个 RLS 档位。`ENABLE` 阻止 anon/authenticated；`FORCE` 也阻止表 OWNER，除非他们持有 BYPASSRLS。我们只使用 `ENABLE`，与 `src/schema.sql`、迁移 v24 和 v29 中的态势一致。`FORCE` 会将非 BYPASSRLS 应用锁定在自己新创建的表之外（触发器函数继承调用者的角色，而不是 gbrain 角色） — 这违背了上面的跨应用共存故事。如果你想在特定的 gbrain 拥有的表上进行纵深防御 `FORCE`，在你自己的迁移中明确添加它；gbrain 的自动 RLS 默认不会让你选择加入。

## 1% 的情况：故意豁免

有时公共表应该对 anon key 可读。支持公共仪表板的分析视图。只读引用表。自带前端并故意使用 anon key 进行读取的插件。

gbrain 为这些情况提供了逃生舱口。设置它是故意痛苦的。这是特性。

### 格式

```sql
-- 在 psql 中，以 BYPASSRLS 角色连接（例如 postgres）：
COMMENT ON TABLE public.your_table IS
  'GBRAIN:RLS_EXEMPT reason=<why this is anon-readable on purpose>';
```

规则：

- 注释值**必须**以 `GBRAIN:RLS_EXEMPT` 开头（区分大小写）。
- 它**必须**包含 `reason=` 后跟至少 4 个字符的理由。
- 没有其他前缀，没有配置文件中的复选框，没有环境变量。只有 Postgres 表注释才算数。
- 如果表上的 RLS 也关闭（为了让 anon key 实际读取必须关闭），你还需要显式运行 `ALTER TABLE ... DISABLE ROW LEVEL SECURITY;`。仅禁用是不够的；注释是告诉 doctor 这是故意的标志。

### 示例

```sql
ALTER TABLE public.expenses_ramp DISABLE ROW LEVEL SECURITY;
COMMENT ON TABLE public.expenses_ramp IS
  'GBRAIN:RLS_EXEMPT reason=analytics-only, anon-readable ok, owner=garry, 2026-04-22';
```

之后，`gbrain doctor` 报告：

```
rls: ok — RLS enabled on 20/21 public tables (1 explicitly exempt: expenses_ramp)
```

请注意，每次后续运行都会按名称重新列出你的豁免。这是故意的。逃生舱口不是一次性签署，而是反复提醒。如果你想知道哪些表是开放的，运行 `gbrain doctor`。

## 为什么使用 SQL 而不是 CLI 子命令

gbrain **不**提供 `gbrain rls-exempt add <table>` 命令。CLI 命令会让代理轻松地静默打开表以进行 anon 读取。psql 注释要求强制操作员在 SQL 中输入理由，这：

- 在 shell 历史中可见。
- 在 git 跟踪的 schema dump 中可见。
- 在下次恢复时的 `pg_dump` 输出中可见。
- 在每次运行的 `gbrain doctor` 输出中可见。

代理**仍然可以**运行 SQL，但它无法在用户看不到操作的情况下执行。这就是"以血书写"的设计。

## 稍后审计豁免

要查看当前数据库中的所有豁免：

```sql
SELECT
  c.relname AS table_name,
  obj_description(c.oid, 'pg_class') AS comment
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'public'
  AND c.relkind = 'r'
  AND obj_description(c.oid, 'pg_class') LIKE 'GBRAIN:RLS_EXEMPT%';
```

如果该列表比你记得签署的要长，那就是信号。

## 删除豁免

只需删除注释并重新启用 RLS：

```sql
ALTER TABLE public.expenses_ramp ENABLE ROW LEVEL SECURITY;
COMMENT ON TABLE public.expenses_ramp IS NULL;
```

`gbrain doctor` 停止将该表列为豁免，并像其他表一样检查它。

## PGLite

如果你使用 PGLite（零配置默认值），doctor 会完全跳过此检查：PGLite 是嵌入式的、单用户的，前面没有 PostgREST。公共模式暴露风险不存在。你会看到：

```
rls: ok — Skipped (PGLite — no PostgREST exposure, RLS not applicable)
```

如果你以后迁移到 Supabase 或自托管的 Postgres，检查会开始运行，并会标记任何没有 RLS 的表。

## 自托管 Postgres

如果你在没有 PostgREST 的情况下运行 Postgres，anon key 暴露不适用。但 gbrain 仍然会在缺少 RLS 时失败检查，因为：

- 框架是"所有公共表都有 RLS"是 gbrain 的安全不变量，而不是特定于 Supabase 的解决方法。
- `ALTER TABLE ... ENABLE RLS` 修复在任何 Postgres 上都是无害的：它只约束非 bypass 角色，gbrain 不使用这些角色。
- 如果你以后在前面放置 PostgREST 或类似工具，防护已经到位。

如果这个框架不适合你的部署，提交问题并提供具体细节，以便我们决定是否需要自托管豁免模式。