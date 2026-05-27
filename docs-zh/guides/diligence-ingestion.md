# 尽职调查摄取：数据室到大脑页面

## 目标

将推介材料、财务模型和数据室材料转换为可搜索、交叉引用的大脑页面，包含看多/看空分析。

## 用户获得的价值

没有这个：推介材料放在电子邮件附件中。财务模型在Google Drive中。与公司大脑页面没有交叉引用。你无法搜索"Acme Corp的A轮融资材料中的关键指标是什么？"

有了这个：每个数据室文档都被提取、整理、交叉引用到公司页面，并且可以搜索。Index.md让你一目了然地看到看多/看空情况。`gbrain query "Acme Corp revenue growth"`找到确切的图表。

## 实现

通过PDF文件名识别数据室材料，包含"Data Deck"、"Intro Deck"、"Data Room"、"Cap Table"、"Financial Model"、"Investor Memo"、"Pitch Deck"或系列轮次名称。包含Revenue、Retention、Cohorts、CAC、Gross Margin、Unit Economics、ARR的电子表格标签。用户语言如"data room"、"diligence"、"deck"、"pitch"、"fundraise materials"。

### 9步管道

**步骤1：识别公司。**
从文档内容或文件名中识别公司名称。
检查`brain/companies/{slug}.md`是否存在。

**步骤2：创建尽职调查目录。**

```bash
mkdir -p brain/diligence/{company-slug}/.raw
```

**步骤3：提取内容。**

- **PDF：** 使用PDF提取工具。对于扫描/图像密集型PDF，使用OCR（例如Mistral OCR或类似工具）。
- **电子表格：** 将每个工作表导出为CSV。对于Google Sheets：
  ```
  https://docs.google.com/spreadsheets/d/{ID}/gviz/tq?tqx=out:csv&sheet={Sheet Name}
  ```

**步骤4：整理并保存。**
将提取的内容写入`brain/diligence/{company}/{doc-name}.md`：
- 文档标题和类型
- 逐节细分，包含关键指标
- 值得注意的脚注或警告
- 相关的原始数据表

**步骤5：保存原始文件。**
将原始PDF/文件复制到`brain/diligence/{company}/.raw/`
保留原始文件以供参考。整理版本用于搜索。

**步骤6：创建或更新index.md。**
每个尽职调查目录都需要一个`index.md`：

```markdown
# {Company Name} — 尽职调查

## 轮次详情
- 阶段：A轮
- 金额：$10M
- 日期：2026-04

## 文档清单
- [推介材料](pitch-deck.md) — 25页，公司概述 + 业务进展
- [财务模型](financial-model.md) — 5个标签页，3年预测
- [股权结构表](cap-table.md) — 当前所有权 + 期权池

## 关键发现
- 收入在过去6个月每月增长30%
- CAC回收期：4个月
- 净留存率：135%

## 看多理由
- 强大的产品市场匹配信号（NPS 72）
- 正在扩展到相邻垂直领域

## 看空理由
- 单一客户占收入的40%
- 烧钱率上季度增加了3倍

## 开放性问题
- 盈利路径是什么？
- 护城河的防御性如何？
```

**步骤7：丰富公司大脑页面。**
更新`brain/companies/{slug}.md`：
- 向前置添加文档来源
- 使用关键发现更新编译真相
- 添加"另见"链接到尽职调查目录
- 如果公司页面不存在，通过丰富技能创建一个

**步骤8：提交。**

```bash
cd brain/ && git add -A && git commit -m "diligence: {Company} — {doc type} ingestion" && git push
```

**步骤9：发布（如要求）。**
当用户想要一个可共享的摘要时，创建一个密码保护的发布版本。删除内部笔记和原始评估语言。

### 质量标准

一个好的尽职调查页面读起来像一份情报评估：
- **他们说什么** vs **数据显示什么**（差距就是洞察力）
- 明确的看多/看空情况（不仅仅是摘要）
- 关键指标突出显示，不被埋没
- 决策前需要回答的开放性问题

## 需要注意的地方

1. **PDF提取是有损的。** 扫描的材料和图像密集型PDF在提取过程中会丢失表格和图表。始终检查整理后的输出与原始`.raw/`文件。如果关键指标缺失，使用OCR重新提取或手动转录。

2. **重新摄取时的幂等性。** 如果用户为同一家公司发送更新的材料，不要创建重复目录。检查现有的`brain/diligence/{company-slug}/`并就地更新。如果应该保留旧版本，在文档文件后追加版本后缀。

3. **index.md完整性。** index.md是整个尽职调查包的入口点。如果缺少看多/看空情况或开放性问题，尽职调查就是不完整的。即使某些部分需要判断，也要始终生成所有部分——明确标记不确定的评估。

## 验证方法

1. **搜索关键指标。** 摄取后，运行
   `gbrain search "revenue growth"` 或 `gbrain search "{company name} CAC"`。
   整理后的内容应该出现在结果中。如果没有，说明同步或嵌入步骤被遗漏了。

2. **检查公司页面交叉引用。** 打开
   `brain/companies/{slug}.md`并验证它链接到尽职调查目录。
   编译真相部分应该包含来自材料的关键发现。

3. **验证index.md有所有部分。** 打开
   `brain/diligence/{company}/index.md`并确认它有Round Details、
   Document Inventory、Key Findings、Bull Case、Bear Case和Open Questions。
   缺少部分意味着管道提前停止。

---

*[GBrain Skillpack](../GBRAIN_SKILLPACK.md) 的一部分。*