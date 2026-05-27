# Takes vs Facts — 架构区别

gbrain 有两个认识论存储层，用于不同的目的。**切勿混淆它们。**

## Takes（冷存储 — `takes` 表）

认识论层。谁相信什么，带有置信权重和时间。

- **来源：** 通过 LLM 分析从大脑页面（markdown）中提取
- **范围：** 多持有者 — 捕获来自*任何*说话者的信念，而不仅仅是大脑所有者
- **种类：** `take`（观点）、`fact`（可验证）、`bet`（预测）、`hunch`（直觉）
- **生命周期：** 冷存储，回顾性。当页面更改或重新提取运行时更新。
- **规模：** 成熟大脑中数千个持有者跨越 100K+ 行

**示例 takes：**
- `holder=people/garry-tan kind=bet` "AI 将在 2030 年前取代 50% 的编码工作" (w=0.75)
- `holder=people/jared-friedman kind=take` "Momo 有很强的留存率" (w=0.80)
- `holder=world kind=fact` "Clipboard 筹集了 1 亿美元 C 轮融资" (w=1.0)
- `holder=brain kind=hunch` "Garry 有英雄/拯救者模式" (w=0.70)

**查询接口：** `gbrain takes list`、`gbrain takes search`、`gbrain think`

## Facts（热内存 — `facts` 表，v0.31）

来自大脑所有者对话的个人知识。实时捕获。

- **来源：** 通过 facts hook（Haiku）从对话中逐轮提取
- **范围：** 单用户 — 仅大脑所有者陈述的知识
- **种类：** `event`、`preference`、`commitment`、`belief`、`fact`
- **生命周期：** 热存储，实时。在对话发生时捕获。
- **桥梁：** 梦境周期的 `consolidate` 阶段每晚将热事实提升为冷 takes

**示例 facts：**
- `kind=event` "我明天与 Brian 有会议"
- `kind=preference` "我不喝咖啡"
- `kind=commitment` "我们决定嵌套监护权"
- `kind=belief` "我认为市场过热"

**查询接口：** `gbrain recall`、MCP `_meta.brain_hot_memory`

## 分类错误

**切勿将 takes 转储到 facts 表中。** Takes 包括其他人的归因信念（Jared 对公司的评估、PG 对学校的看法、创始人的收入声明）。这些**不是**大脑所有者的个人事实。

**切勿未经转换将 facts 转储到 takes 表中。** Facts 的范围限于所有者在对话中所说的内容。它们只有通过梦境周期的 consolidate 阶段才能成为 takes，该阶段添加了适当的归因、去重和时间推理。

## 桥梁

梦境周期的 `consolidate` 阶段（v0.31）是单向桥梁：

```
hot facts → [dream consolidate] → cold takes
```

事实流向**一个**方向。consolidate 阶段：
1. 按实体分组相关事实
2. 对现有 takes 进行去重
3. 将持久事实提升为具有适当 holder/weight 的 takes
4. 用 `consolidated_at` + `consolidated_into` 标记已合并的事实

## 生产提取数据（2026-05-10）

首次对约 100K 页面大脑进行完整 takes 提取运行：
- **模型：** Azure GPT-5.5（成本为 Opus 的 1/8 时达到 Opus 质量 — $0.033 vs $0.260/page）
- **结果：** 从 28,256 个磁盘页面提取 100,720 个 takes，花费 $361.49，83 个错误（0.3%）
- **细分：** 70,960 takes / 24,342 facts / 2,875 bets / 2,649 hunches
- **持有者：** 6,239 个唯一持有者
- **跨模态评估：** 总体 6.8/10（GPT-5.5 + Opus 4.6 独立评分）

### 评估维度

| 维度 | 分数 | 说明 |
|-----------|-------|-------|
| 准确性 | 7.5 | 声明忠实地代表来源 |
| 归因 | 6.5 | 持有者/主体混淆是 #1 问题 |
| 权重校准 | 7.0 | 范围使用良好，一些假精度 |
| 种类分类 | 6.5 | 偶尔的 fact/take 误分类 |
| 信号密度 | 6.5 | 一些琐碎的提取通过 |

### 提取提示的关键经验

1. **持有者 ≠ 主体。** "Garry 有英雄/拯救者模式" → holder=brain，NOT people/garry-tan
2. **原子声明。** 将复合声明拆分为单独的行
3. **放大 ≠ 认可。** 仅转发 → 最大权重 0.55
4. **自我报告 ≠ 已验证。** "报告 7 位数" → holder=person，weight=0.75，NOT world/1.0
5. **不要假精度。** 使用 0.05 增量（0.35、0.55、0.75），而不是 0.74 或 0.82
6. **"那又怎样"测试。** 跳过 Twitter 句柄、关注者数量、明显的元数据