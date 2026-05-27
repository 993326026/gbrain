# 实时同步：保持索引最新

## 目标

大脑仓库中的每个markdown更改都能在几分钟内自动搜索到，无需手动干预。

## 用户获得的价值

没有这个：你在大脑页面中纠正了一个幻觉，但向量数据库继续提供旧文本，因为没有人运行`gbrain sync`。陈旧的搜索结果会侵蚀信任。大脑变得不可靠。

有了这个：编辑内容在几分钟内出现在搜索结果中。向量数据库自动保持与大脑仓库同步。你永远不必记得运行sync。

## 实现

### 前提条件：Session Mode池化器

Sync在每次导入时使用`engine.transaction()`。如果`DATABASE_URL`指向Supabase的**Transaction mode**池，sync将抛出`.begin() is not a function`并**静默跳过大多数页面**。这是"sync运行但什么都没发生"的首要原因。

修复：使用**Session mode**池字符串（端口6543，Session mode）或直接连接（端口5432，仅IPv6）。通过运行`gbrain sync`并检查`gbrain stats`中的页面计数是否与仓库中的可同步文件计数匹配来验证。

### 基础命令

始终链式运行sync + embed：

```bash
gbrain sync --repo /path/to/brain && gbrain embed --stale
```

- `gbrain sync --repo <path>` -- 一次性增量同步。通过`git diff`检测更改，仅导入更改的内容。对于小更改集（<= 100个文件），嵌入在导入期间内联生成。
- `gbrain embed --stale` -- 为任何没有嵌入的块回填嵌入。大同步（>100个文件）或先前`--no-embed`运行的安全网。
- `gbrain sync --watch --repo <path>` -- 前台轮询循环，每60秒一次（可通过`--interval N`配置）。小更改集内联嵌入。连续5次失败后退出，因此在进程管理器下运行或与cron回退配对。

### 方法1：定时任务（推荐）

每5-30分钟运行一次。适用于任何cron调度器。

```bash
gbrain sync --repo /data/brain && gbrain embed --stale
```

**OpenClaw:**
```
Name: gbrain-auto-sync
Schedule: */15 * * * *
Prompt: "Run: gbrain sync --repo /data/brain && gbrain embed --stale
  Log the result. If sync fails with .begin() is not a function,
  the DATABASE_URL is using Transaction mode pooler."
```

**Hermes:**
```
/cron add "*/15 * * * *" "Run gbrain sync --repo /data/brain &&
  gbrain embed --stale. Log the result." --name "gbrain-auto-sync"
```

### 方法2：长期观察器

用于近乎即时同步（60秒轮询）。在自动重启退出的进程管理器下运行。由于`--watch`在重复失败时退出，因此与cron回退配对。

```bash
gbrain sync --watch --repo /data/brain
```

### 方法3：Git钩子/ Webhook

在push事件时触发同步以实现即时同步（<5秒）。

- **GitHub webhook:** 设置webhook调用`gbrain sync --repo /data/brain && gbrain embed --stale`。验证`X-Hub-Signature-256`与共享密钥。
- **Git post-receive hook:** 如果大脑仓库在同一台机器上。

### 同步内容

Sync只索引"可同步的"markdown文件。以下内容按设计排除：
- 隐藏路径（`.git/`、`.raw/`等）
- `ops/`目录
- 元文件：`README.md`、`index.md`、`schema.md`、`log.md`

### Sync是幂等的

并发运行是安全的。同一提交上的两次同步是no-op，因为内容哈希匹配。如果cron和`--watch`同时触发，不会有冲突。

## 需要注意的地方

1. **始终链式运行sync + embed。** 运行`gbrain sync`而不运行`gbrain embed --stale`会使新块没有嵌入。它们存在于数据库中但对向量搜索不可见。始终一起运行两个命令。`&&`确保只有在sync成功时才运行embed。

2. **--watch轮询，不流式传输。** `--watch`标志每60秒轮询一次（可配置）。它不是文件系统监视器或git钩子。连续5次失败后退出，因此需要进程管理器（systemd、pm2）或cron回退才能保持运行。不要假设它永远运行。

3. **Webhook需要服务器运行。** 如果你使用GitHub webhook进行即时同步，接收服务器必须正在运行且可访问。如果服务器在push时宕机，同步将被错过。将webhook与cron回退配对，以捕获webhook错过的任何内容。

## 验证方法

1. **编辑文件并搜索更改。** 编辑大脑markdown文件，提交并推送。等待下一个同步周期（cron间隔或`--watch`轮询）。运行`gbrain search "<text from the edit>"`。更新的内容应该出现在结果中。如果返回旧内容，同步失败。

2. **比较页面计数与文件计数。** 运行`gbrain stats`并计算大脑仓库中可同步的markdown文件数量。数据库中的页面计数应该匹配。如果它们不一致，文件被静默跳过（可能是Transaction mode池问题）。

3. **检查嵌入块计数。** 在`gbrain stats`中，嵌入块计数应该接近总块计数。较大的差距意味着`gbrain embed --stale`在sync后没有运行，使块对向量搜索不可见。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*