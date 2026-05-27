# 嵌入提供商

GBrain 附带 14 个嵌入提供商配方，涵盖 OpenAI、主要托管替代方案、三个本地选项和一个通用逃生舱口（LiteLLM 代理）。运行 `gbrain providers list` 查看实时注册表；`gbrain providers explain --json` 为代理发出机器可读矩阵。

本页面是人类可读的对应内容：每个提供商的能力、环境变量设置、维度、成本和已知约束。

## 快速开始

```
gbrain providers list                          # 查看所有提供商
gbrain providers env <provider-id>             # 查看所需的环境变量
gbrain providers test --model openai:text-embedding-3-large   # 烟雾测试
gbrain init --pglite --model voyage            # 使用非默认提供商
```

## 摘要表

| 提供商 | 环境变量 | 默认维度 | 成本 ($/1M tokens) | 本地？ | 多模态？ |
|---|---|---|---|---|---|
| `openai` | `OPENAI_API_KEY` | 1536 | 0.13 | 否 | 否 |
| `voyage` | `VOYAGE_API_KEY` | 1024 | 0.18 | 否 | 是 (`voyage-multimodal-3`) |
| `google` | `GOOGLE_GENERATIVE_AI_API_KEY` | 768 | 0.025 | 否 | 否 |
| `azure-openai` | `AZURE_OPENAI_API_KEY`, `AZURE_OPENAI_ENDPOINT`, `AZURE_OPENAI_DEPLOYMENT` | 1536 | 0.13 | 否 | 否 |
| `minimax` | `MINIMAX_API_KEY` | 1536 | 0.07 | 否 | 否 |
| `dashscope` | `DASHSCOPE_API_KEY` | 1024 | 变化 | 否 | 否 |
| `zhipu` | `ZHIPUAI_API_KEY` | 1024 | 变化 | 否 | 否 |
| `ollama` | (无 — 本地运行) | 768 | 0 | 是 | 否 |
| `llama-server` | (无 — 本地运行) | 用户设置 | 0 | 是 | 否 |
| `litellm` | `LITELLM_API_KEY` (可选) | 用户设置 | 变化 | 是 (代理) | 否 |
| `together` | `TOGETHER_API_KEY` | 768 | 变化 | 否 | 否 |
| `anthropic` | (无嵌入模型 — 仅聊天) | — | — | — | — |
| `deepseek` | (无嵌入模型 — 仅聊天) | — | — | — | — |
| `groq` | (无嵌入模型 — 仅聊天) | — | — | — | — |

## 决策树

- **成本敏感，仅英语**：Ollama（免费，本地）或 Voyage（付费，性价比最佳）。
- **质量优先**：Voyage `voyage-4-large`（1024-2048 维度，比 OpenAI tiktoken 密集约 3-4 倍）。
- **重排序配对**：Voyage（其重排序器 `rerank-2.5` 与 Voyage 嵌入完美配对）。
- **企业合规**：Azure OpenAI（数据驻留 + 私有端点）或通过 llama-server / Ollama 自托管。
- **中国地区**：DashScope（阿里巴巴）或 Zhipu（BigModel）。DashScope 的国际端点在 `dashscope-intl.aliyuncs.com`；覆盖 `provider_base_urls.dashscope` 以使用中国端点。
- **开源本地，完全控制**：llama-server（`llama.cpp`）用于任何 GGUF 模型；Ollama 用于精选目录。
- **其他任何情况**：LiteLLM 代理。在任何提供商（Bedrock、Vertex、Cohere、Jina、Fireworks 等）前面运行 LiteLLM，并通过 `LITELLM_BASE_URL` 将 gbrain 指向它。

## 各提供商详情

### OpenAI

默认。设置 `OPENAI_API_KEY`。模型：`text-embedding-3-large`（最大 3072，默认 1536），`text-embedding-3-small`（1536）。通过 `dimensions` 字段支持 Matryoshka — gbrain 从 `embedding_dimensions` 配置固定它，因此现有的 1536 维大脑在 SDK 升级后保持对齐。

### Voyage AI

Voyage 4 系列（2026年1月发布）的一流质量。设置 `VOYAGE_API_KEY`。模型：`voyage-4-large`、`voyage-4`、`voyage-4-lite`、`voyage-4-nano`、`voyage-3.5`、`voyage-code-3`（代码调优）、`voyage-finance-2`、`voyage-law-2`、`voyage-multimodal-3`（文本 + 图像）。

Voyage 4 系列在所有变体之间共享嵌入空间，因此你可以用 `voyage-4-large` 索引并用 `voyage-4-lite` 查询，无需重新索引。维度：256、512、1024、2048。**2048 超过 pgvector 的 HNSW 上限 2000** — 这些大脑回退到精确向量扫描（仍然正确，只是较慢）。

### Google Gemini

设置 `GOOGLE_GENERATIVE_AI_API_KEY`（AI Studio 公共 API 密钥）。模型：`gemini-embedding-001`。默认 768 维度；Matryoshka 最高 3072。便宜。

对于 GCP 服务账号 / Vertex AI 认证（生产部署），请参阅 v0.32.x 后续版本 — Vertex ADC 已在路线图上。

### Azure OpenAI

Azure 租户后面的企业级 OpenAI。必需环境：`AZURE_OPENAI_API_KEY`、`AZURE_OPENAI_ENDPOINT`（例如 `https://my-resource.openai.azure.com`）、`AZURE_OPENAI_DEPLOYMENT`（Azure 门户中的部署名称）。可选：`AZURE_OPENAI_API_VERSION`（默认为 `2024-10-21`）。

与普通 OpenAI 不同，Azure 使用 `api-key:` 标头（不是 `Authorization: Bearer`）和带 `?api-version=` 查询参数的模板 URL — gbrain 通过配方的 resolveAuth + resolveOpenAICompatConfig 覆盖处理两者。

模型：`text-embedding-3-large`、`text-embedding-3-small`、`text-embedding-ada-002`（你的 Azure 部署必须提供请求的模型）。

### MiniMax（海螺AI）

设置 `MINIMAX_API_KEY`。可选 `MINIMAX_GROUP_ID` 用于组织范围的账户。模型：`embo-01`（1536 维度）。

MiniMax 的 API 使用 `type: 'db' | 'query'` 字段进行非对称检索。v0.32 将所有内容路由为 `type='db'`（对称检索 — 索引和查询使用相同的向量空间）。非对称查询支持是 v0.32.x 后续版本。

### DashScope（阿里巴巴）

设置 `DASHSCOPE_API_KEY`。默认国际端点在 `dashscope-intl.aliyuncs.com`；覆盖 `provider_base_urls.dashscope` 以使用中国端点。模型：`text-embedding-v3`（当前；Matryoshka 64-1024 维度）、`text-embedding-v2`。

CJK 主导内容的令牌化比 OpenAI tiktoken 更密集；gbrain 声明 `chars_per_token: 2`，因此批处理预分割留有空间。

### Zhipu AI（BigModel）

设置 `ZHIPUAI_API_KEY`。模型：`embedding-3`（当前；Matryoshka 256-2048 维度）、`embedding-2`。v0.32 默认值为 1024（HNSW 兼容）。2048 维度选项有效但会落入精确扫描分支（见上面的 Voyage 4 Large 注释）。

### Ollama（本地）

无需环境变量 — Ollama 在本地无认证运行。可选 `OLLAMA_BASE_URL`（默认 `http://localhost:11434/v1`）和 `OLLAMA_API_KEY`（用于启用认证的部署）。

配方附带 `nomic-embed-text`（768d，推荐）、`mxbai-embed-large`（1024d）、`all-minilm`（384d）。`gbrain providers test --model ollama:nomic-embed-text` 对本地安装进行烟雾测试。

### llama-server（本地，llama.cpp）

`llama.cpp` 的 `llama-server --embeddings` 端点。无需环境变量。可选 `LLAMA_SERVER_BASE_URL`（默认 `http://localhost:8080/v1`）和 `LLAMA_SERVER_API_KEY`。

用户驱动的模型：用 `--model <gguf-path> --embeddings` 启动 llama-server，然后运行 `gbrain init --embedding-model llama-server:<your-id> --embedding-dimensions <N>`。配方拒绝隐式简写 `--model llama-server`，因为没有规范的第一个模型。

### LiteLLM 代理（通用逃生舱口）

在任何提供商前面运行 [LiteLLM](https://docs.litellm.ai/docs/proxy/quick_start) — Bedrock、Vertex、Cohere、Jina、Fireworks、OctoAI 等。代理将所有内容标准化为 OpenAI 兼容的 API；gbrain 通过 `LITELLM_BASE_URL` 指向代理并代理调用。

这是"我的提供商不在上面列表中"的万能解决方案。设置 LiteLLM，然后 `gbrain init --embedding-model litellm:<your-model-id> --embedding-dimensions <N>`。

## 选择维度

三个数字很重要：
1. **提供商的原生维度**：每个模型有一个"真实"输出维度（例如 OpenAI `text-embedding-3-large` 是 3072 原生）。
2. **Matryoshka 缩减**：大多数现代提供商允许你通过 `dimensions` 字段请求更小的向量。
3. **HNSW 上限**：pgvector 的 HNSW 索引支持最多 2000 维度。超过此限制的大脑回退到精确向量扫描（较慢但正确；gbrain 通过 `src/core/vector-index.ts` 中的 `chunkEmbeddingIndexSql` 自动处理 SQL）。

对于大多数用户：**保持在 1024 或 1536**。在噪声底以下，更大并不更好；更小可以节省磁盘 + RAM，对 Matryoshka 提供商的召回率损失很小。

## 我的提供商不在列表中

三个选项：

1. **使用 LiteLLM 代理**（上面）— 通用逃生舱口。适用于 100+ 提供商。
2. **提交功能请求** 在 [github.com/garrytan/gbrain/issues](https://github.com/garrytan/gbrain/issues)，提供提供商的 API 文档 URL 和设置片段。配方约 30-40 行 TypeScript。
3. **提交配方**：克隆，复制 `src/core/ai/recipes/voyage.ts` 作为黄金标准的 openai-compat 模板，在 `src/core/ai/recipes/index.ts` 注册，在 `test/ai/recipe-<name>.test.ts` 下添加每个配方的烟雾测试。配方契约测试（`test/ai/recipes-contract.test.ts`）和铁规则回归测试固定结构不变量。

## 在现有大脑上切换提供商

嵌入维度在 `gbrain init` 时被烘焙到架构中。要在初始化后更改提供商，通常需要重新嵌入：

1. 更新配置：`gbrain config set embedding_model <provider>:<model>` 和 `embedding_dimensions <N>`。
2. 如果维度更改，重新索引架构：`gbrain doctor` 将检测不匹配并打印确切的 `ALTER TABLE` 配方。
3. 重新嵌入：`gbrain embed --all`（或 `--stale` 进行增量）。

`gbrain doctor` 8c "alternative_providers" 显示已配置环境但未配置的提供商 — 当你配置了 OpenAI 但也导出了例如 `VOYAGE_API_KEY` 并想知道无需额外设置即可切换时很有用。