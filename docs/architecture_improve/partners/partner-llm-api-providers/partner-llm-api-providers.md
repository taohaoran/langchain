# partner-llm-api-providers 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文只展开 7 家云端 LLM API 提供商集成的职责边界，不重复展开 OpenAI/Anthropic 主适配层、
> 向量库、本地推理、Exa 搜索等相邻叶子。
>
> 源码基准：`libs/partners/`（groq/mistralai/openrouter/deepseek/xai/perplexity/fireworks 七包）；branch `master`，commit `89252a8f`。

## 1. 功能清单

本叶子覆盖 7 个独立集成包，分两种实现模式：

### 模式 A：复用 OpenAI 适配层（继承 `BaseChatOpenAI`）

| 包 | 主类 | 位置 | 说明 |
|---|---|---|---|
| deepseek | `ChatDeepSeek(BaseChatOpenAI)` | `deepseek/langchain_deepseek/chat_models.py:110` | DeepSeek 兼容 OpenAI 接口，仅改 base_url/api_key 环境变量 |
| xai | `ChatXAI(BaseChatOpenAI)` | `xai/langchain_xai/chat_models.py:61` | xAI Grok 兼容 OpenAI 接口，改 base_url/key 环境变量 |

### 模式 B：自实现 `BaseChatModel`（独立适配）

| 包 | 主类 | 位置 | 额外模块 |
|---|---|---|---|
| groq | `ChatGroq(BaseChatModel)` | `groq/langchain_groq/chat_models.py:126` | 无 |
| mistralai | `ChatMistralAI(BaseChatModel)` | `mistralai/langchain_mistralai/chat_models.py:619` | `MistralAIEmbeddings`（embeddings.py:59） |
| openrouter | `ChatOpenRouter(BaseChatModel)` | `openrouter/langchain_openrouter/chat_models.py:106` | 统一网关接数百模型 |
| perplexity | `ChatPerplexity(BaseChatModel)` | `perplexity/langchain_perplexity/chat_models.py:617` | `PerplexitySearchRetriever`（retrievers.py:13）、`PerplexitySearchResults`（tools.py:12） |
| fireworks | `ChatFireworks(BaseChatModel)` | `fireworks/langchain_fireworks/chat_models.py:799` | `FireworksRerank`（rerank.py:24）、`FireworksEmbeddings`（embeddings.py:8） |

每个包均含 `data/_profiles.py`（模型能力 profile）与 `_version.py`。

**深读代表**：groq（模式 B 自实现代表）、deepseek/xai（模式 A 复用代表）、openrouter（统一网关）。
**共性列表**：mistralai/perplexity/fireworks 的 `_generate`/`_stream`/`bind_tools` 结构与 groq 同构（消息→厂商 SDK→API→响应转换），未逐行深读，如实披露缺口。

## 2. 核心类型与接口清单

| 类型 | 位置 | 职责 |
|---|---|---|
| `ChatDeepSeek(BaseChatOpenAI)` | deepseek chat_models.py:110 | 复用 OpenAI 适配层，覆写 `_default_params` 设 base_url 为 DeepSeek 端点 |
| `ChatXAI(BaseChatOpenAI)` | xai chat_models.py:61 | 复用 OpenAI 适配层，base_url 指向 xAI 端点 |
| `ChatGroq(BaseChatModel)` | groq chat_models.py:126 | 自实现：`_generate`（:644）/`_stream`（:691）/`bind_tools`（:901）/`with_structured_output`（:950） |
| `ChatOpenRouter(BaseChatModel)` | openrouter chat_models.py:106 | 自实现：统一 API 网关，支持数百模型路由 |
| `ChatMistralAI(BaseChatModel)` | mistralai chat_models.py:619 | 自实现 + `MistralAIEmbeddings` |
| `ChatPerplexity(BaseChatModel)` | perplexity chat_models.py:617 | 自实现 + 搜索检索器/工具 |
| `ChatFireworks(BaseChatModel)` | fireworks chat_models.py:799 | 自实现 + rerank/embeddings |
| `FireworksRerank(BaseDocumentCompressor)` | fireworks rerank.py:24 | 文档重排序压缩器 |
| `PerplexitySearchRetriever(BaseRetriever)` | perplexity retrievers.py:13 | Perplexity 搜索结果转 Document |

**依赖倒置**：所有 Chat 类均实现 `langchain_core.BaseChatModel`；embeddings 实现 `Embeddings`；rerank 实现 `BaseDocumentCompressor`；retriever 实现 `BaseRetriever`。

## 3. 关键调用链

### 链 1：模式 A（deepseek/xai）——复用 OpenAI 适配层

1. `ChatDeepSeek`（deepseek chat_models.py:110）继承 `BaseChatOpenAI`（见 partner-openai 叶子）。
2. 仅覆写构造：`openai_api_base` 默认指向 DeepSeek 端点（`https://api.deepseek.com`），`openai_api_key` 从 `DEEPSEEK_API_KEY` 环境变量读取。
3. `_generate`/`_stream`/消息转换/工具调用/结构化输出全部复用 `BaseChatOpenAI` 实现，零重写。

### 链 2：模式 B（groq）——自实现

1. `ChatGroq._generate`（groq chat_models.py:644）：把 langchain messages 转为 groq SDK 的 chat completion 请求格式。
2. 调 `self.client.chat.completions.create(...)`（groq SDK），SDK 发 HTTPS 到 Groq 云端 API。
3. 响应经转换为 langchain `ChatResult`（AIMessage）。
4. `_stream`（:691）对称实现流式 chunk。

### 链 3：OpenRouter 统一网关

1. `ChatOpenRouter`（openrouter chat_models.py:106）自实现 `BaseChatModel`。
2. 请求发往 OpenRouter 统一端点，由 OpenRouter 路由到后端实际模型提供商（OpenAI/Anthropic/Google 等）。
3. 支持通过 `model` 参数指定 `provider/model` 格式路由。

## 4. 配置项

| 配置项 | 说明 |
|---|---|
| API key 环境变量 | `DEEPSEEK_API_KEY` / `XAI_API_KEY` / `GROQ_API_KEY` / `MISTRAL_API_KEY` / `OPENROUTER_API_KEY` / `PERPLEXITY_API_KEY` / `FIREWORKS_API_KEY` |
| `model` / `model_name` | 模型名（各包必填） |
| `base_url` / `api_base` | 各包默认指向各自云端端点 |
| `max_retries` / `timeout` | 透传到底层 SDK |
| groq 特有 | `GROQ_API_BASE` 环境变量覆盖 base_url（chat_models.py:440） |

## 5. 错误与重试语义

- **模式 A**：错误映射完全复用 `BaseChatOpenAI` 的 `Model*Error` 体系（见 partner-openai 叶子 §5）。
- **模式 B**：各包自行把厂商 SDK 异常映射为 langchain_core `Model*Error` 族；重试由厂商 SDK 负责。
- **重试**：统一由底层厂商 SDK 客户端负责，本层不实现应用层重试队列。

## 6. 并发细节

- **同步/异步双实现**：各 `_generate`/`_stream` 与 `_agenerate`/`_astream` 对称成对。
- **客户端懒初始化**：`client`/`async_client` 在 `validate_environment` 时构建。
- **模式 A 零额外并发逻辑**：直接复用 `BaseChatOpenAI` 的并发模型。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- 7 个包各自的 `chat_models.py`：Chat 模型适配
- mistralai `embeddings.py`：Mistral 嵌入
- fireworks `rerank.py`/`embeddings.py`：重排序与嵌入
- perplexity `retrievers.py`/`tools.py`：搜索检索器与工具
- 各包 `data/_profiles.py`：模型 profile

**Out-of-Scope（不在本仓库源码内）**

- 7 家云端 API 服务端——外部服务
- 各厂商 Python SDK（openai/groq/mistralai 等）——依赖项
- OpenRouter 后端路由的数百个第三方模型服务端——外部服务
- 相邻叶子：OpenAI/Anthropic 主适配层、向量库、本地推理

## 8. 与相邻子系统交互

- **上游 → 本叶子**：用户应用通过 `BaseChatModel`/`Embeddings`/`BaseRetriever`/`BaseDocumentCompressor` 抽象调用各集成类。
- **本叶子 → 下游**：各 Chat 类 → 厂商 SDK → 云端 API 服务端（外部）。
- **同域交互**：deepseek/xai 反向依赖 partner-openai 的 `BaseChatOpenAI`；其余包独立适配 `langchain_core`。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：核心能力缝是 `BaseChatModel` 抽象的多实现矩阵——同一 `BaseChatModel` 契约下 7 个包提供 7 种云端实现。这是纯 Python 框架的"多后端矩阵"模式。
- **双轨实现**：模式 A（复用 `BaseChatOpenAI`，deepseek/xai）与模式 B（自实现 `BaseChatModel`，其余 5 家）是同一能力的两条实现轨——OpenAI 兼容的提供商直接继承复用，不兼容的独立实现。
- **注册表与工厂**：各包通过 `__init__.py` 直接导出类，用户显式实例化；无字符串→类注册表。
- **可选依赖**：各厂商 SDK 是对应包的硬依赖；API key 通过环境变量注入。
- **配置驱动**：模式 A 中 `openai_api_base` 配置决定请求发往哪个端点——同一 `BaseChatOpenAI` 代码经不同配置指向不同提供商。
- **外部边界**：7 家云端服务端与各厂商 SDK 均标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 云端 API 提供商架构图 | `partner-llm-api-providers-architecture.html` | architecture | standard |
| 调用时序图 | `partner-llm-api-providers-sequence.html` | sequence | **showcase** |

**档位说明**：架构图因跨"langchain_core → 两种继承模式 → 7 个具体类 → 云端服务"四层、10+ 组件、跨层连线较多，showcase 布局校验未一次通过，降为 standard 渲染（修复动作：删减冗余连线至 11 条、缩短标签、合并同类组件）。时序图去掉自调用消息后一次通过 showcase。**缺口披露**：mistralai/perplexity/fireworks 三家的 `_generate` 内部消息转换细节未逐行深读（与 groq 同构），以共性列表说明。JSON IR 源文件位于 `json/` 目录。
