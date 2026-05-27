# `gbrain eval takes-quality` — 可重现的跨模态质量评估

v0.32+ 为 takes 层提供了一个可用于 CI 的质量门。三个前沿模型根据 5 维评分标准对 takes 样本进行评分，运行器聚合为 PASS / FAIL / INCONCLUSIVE，并且收据持久化到 `eval_takes_quality_runs`，以便后续的 `trend` 或 `regress` 可以与历史进行比较。

本文档是消费者契约。兄弟 [gbrain-evals](https://github.com/garrytan/gbrain-evals) 仓库和任何未来的 CI 门读取的收据形状完全如下所示的 JSON。字段在 `schema_version: 1` 时是增量稳定的。破坏性形状更改会增加版本。

## 子命令

| 命令 | 需要大脑？ | 退出码 |
|---|---|---|
| `gbrain eval takes-quality run [flags]` | yes (samples takes) | 0 PASS, 1 FAIL, 2 INCONCLUSIVE |
| `gbrain eval takes-quality replay <receipt>` | **no** (disk-only) | 0 PASS, 1 FAIL, 2 INCONCLUSIVE |
| `gbrain eval takes-quality trend [flags]` | yes (reads runs table) | 0 |
| `gbrain eval takes-quality regress --against <receipt>` | yes | 0 OK, 1 regression |

`replay` 是唯一不需要 `DATABASE_URL` 运行的模式 —— 它从磁盘读取收据文件并重新渲染。其他模式需要大脑。

## `run` 标志

| 标志 | 默认值 | 说明 |
|---|---|---|
| `--limit N` | 100 | 大脑中 N 个 takes 的随机样本。 |
| `--cycles N` | 3 (TTY) / 1 (non-TTY) | 放弃前最多 N 次面板调用；在 PASS 或 INCONCLUSIVE 时提前停止。 |
| `--budget-usd N` | unset | 在下次调用的预计成本超过上限之前中止。`pricing.ts` 中没有条目的模型会大声失败（codex #4）。 |
| `--source db|fs` | `db` | `fs` 保留用于 v0.33+。 |
| `--slug-prefix P` | unset | 将 takes 过滤到 slug 以 P 开头的页面。 |
| `--models a,b,c` | `openai:gpt-4o,anthropic:claude-opus-4-7,google:gemini-1.5-pro` | 逗号分隔的面板。 |
| `--json` | off | 将完整收据输出到 stdout。 |

## 收据 JSON 形状 (`schema_version: 1`)

```json
{
  "schema_version": 1,
  "ts": "2026-05-09T22:00:00.000Z",
  "rubric_version": "v1.0",
  "rubric_sha8": "abcd1234",
  "corpus": {
    "source": "db",
    "n_takes": 100,
    "slug_prefix": null,
    "corpus_sha8": "abcd1234"
  },
  "prompt_sha8": "abcd1234",
  "models_sha8": "abcd1234",
  "models": ["openai:gpt-4o", "anthropic:claude-opus-4-7", "google:gemini-1.5-pro"],
  "cycles_run": 3,
  "successes_per_cycle": [3, 3, 2],
  "verdict": "pass",
  "scores": {
    "accuracy":            { "mean": 7.8, "min": 7, "max": 9, "scores": [9,7,7], "per_model": {...} },
    "attribution":         { "mean": 7.0, "min": 7, "max": 7, "scores": [7,7,7], "per_model": {...} },
    "weight_calibration":  { "mean": 7.5, "min": 7, "max": 8, "scores": [8,7,7], "per_model": {...} },
    "kind_classification": { "mean": 7.2, "min": 7, "max": 8, "scores": [7,8,7], "per_model": {...} },
    "signal_density":      { "mean": 7.0, "min": 6, "max": 8, "scores": [8,7,6], "per_model": {...} }
  },
  "overall_score": 7.3,
  "cost_usd": 1.85,
  "improvements": ["..."],
  "errors": [],
  "verdictMessage": "PASS: every dim mean >=7 and min >=5 ..."
}
```

### 字段参考

- `schema_version` — 锁定契约。添加可选字段是增量的且兼容。重命名、删除或更改语义会增加版本。
- `rubric_version` + `rubric_sha8` — 按评分标准时期隔离趋势行（codex review #3）。当评分标准定义更改时，两个字段都会更新，趋势模式相应地对运行进行分组，因此更严格的评分标准不会静默地看起来像质量下降。
- `corpus.corpus_sha8` — 法官看到的连接 takes-text 的指纹。确定两次运行是否在"相同"样本上。
- `models_sha8` — 排序后的模型 ID 列表上的指纹。在 `--models` 中重新排序模型不会改变 sha（排序是稳定的）。
- `successes_per_cycle` — 每个周期贡献模型的计数。当 (a) 其 JSON 解析 AND (b) 每个声明的评分标准维度都有有限分数时，模型贡献（codex review #5 — 缺失维度会删除贡献）。
- `verdict` — 如果每个维度平均值 >= 7 AND 贡献模型的每个维度最小值 >= 5，则为 `pass`；否则为 `fail`；如果少于 2/3 的模型贡献了完整分数，则为 `inconclusive`。
- `cost_usd` — 通过 `pricing.ts` 的每次调用成本之和。设置 `--budget-usd` 时未知模型会在任何调用触发前产生 `PricingNotFoundError`。

## 收据持久化

收据持久化到 **`eval_takes_quality_runs`**（每个 codex review #6 的数据库权威）AND 磁盘上的 `~/.gbrain/eval-receipts/takes-quality-<corpus>-<prompt>-<models>-<rubric>.json` 作为尽力而为的工件。数据库行在 `receipt_json` JSONB 列中携带完整的收据 JSON，因此当磁盘工件消失时，`replay` 仍然可以通过 `loadReceiptFromDb`（v0.33+ 标志接线）重建。

4-sha 主键是唯一的（`UNIQUE` 约束），因此重新运行相同的评估是 `INSERT ... ON CONFLICT DO NOTHING` —— 幂等性。

## 趋势输出

纯文本（默认）：

```
ts                   rubric  verdict       overall  cost     corpus
─────────────────────────────────────────────────────────────────────────────
2026-05-09T22:00:00  v1.0    pass             7.3   $1.85   abcd1234
2026-05-08T18:30:00  v1.0    fail             6.8   $1.92   ef567890
```

JSON 形状（`--json`）：

```json
{
  "schema_version": 1,
  "rows": [
    { "id": 42, "ts": "...", "rubric_version": "v1.0", "verdict": "pass",
      "overall_score": 7.3, "cost_usd": 1.85, "corpus_sha8": "abcd1234" }
  ]
}
```

## Regress: 在质量上设置 CI 门

```bash
# 捕获基线。
gbrain eval takes-quality run --limit 100 --json \
  > .ci/takes-quality-baseline.json

# 稍后，更改提取提示后：
gbrain eval takes-quality regress --against .ci/takes-quality-baseline.json \
  --threshold 0.5
# exit 0 → 没有超过阈值的回归
# exit 1 → 某些维度下降 > 0.5；CI 失败
```

阈值是算作回归的每个维度平均值下降。默认为 0.5。Regress 重用与先前收据相同的**模型面板 + slug 前缀 + 源**进行苹果对苹果比较。`corpus_sha8` / `prompt_sha8` / `rubric_sha8` 中的差异显示为信息性警告（运行器不会拒绝 —— 这是调用者的决定）。

## 契约稳定性

上面的形状是下游消费者的读取契约。未列出的任何内容（例如内部聚合器状态、网关 providerMetadata）**不在**收据中，可能会未经通知更改。

当您需要演进 schema 时：
1. 增量可选字段 → 不增加版本；旧消费者忽略新键，新消费者读取它。
2. 重命名或删除字段，或更改语义 → 将 `schema_version` 增加到 `2`；运行器在一个版本中发出两种形状作为弃用过渡期。