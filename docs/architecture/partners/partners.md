# partners 域总览

> 本文是 `partners` 域的总览文档。本域归并 LangChain monorepo 中 `libs/partners/` 下的全部第三方集成包分析。
> 系统级总览见 `../system-overview.md`（位于输出根）。

## 域职责概述

`partners` 域是 LangChain 的**集成适配层**：把第三方 LLM 提供商、向量数据库、搜索 API 等外部系统适配到 `langchain_core` 定义的统一抽象接口上。本域不承载实际计算——所有推理、向量检索、搜索均发生在外部服务端或用户注入的组件中；本仓库价值在于**统一接口 + 消息/数据格式转换 + 错误映射 + 懒初始化**。

本域覆盖 `libs/partners/` 下 17 个集成包，约 160 个 Python 源文件 / 5.7 万行。按功能归并为 5 个叶子子系统。

## 叶子索引表

| 叶子 ID | 叶子名称 | 覆盖包 | 核心能力 | 设计文档 |
|---|---|---|---|---|
| partner-openai | OpenAI 集成 | `openai/`（84 py，最大） | ChatOpenAI / AzureChatOpenAI / OpenAIEmbeddings / 工具 / 中间件 | [partner-openai.md](partner-openai/partner-openai.md) |
| partner-anthropic | Anthropic Claude 集成 | `anthropic/`（41 py） | ChatAnthropic / 工具中间件族 / Prompt 缓存 | [partner-anthropic.md](partner-anthropic/partner-anthropic.md) |
| partner-vector-stores | 向量数据库集成 | `qdrant/`（38 py）+ `chroma/`（14 py） | QdrantVectorStore / Chroma / 稀疏向量 / 图像检索 | [partner-vector-stores.md](partner-vector-stores/partner-vector-stores.md) |
| partner-model-providers | 其他模型提供商 | `perplexity`/`fireworks`/`groq`/`mistralai`/`openrouter`/`xai`/`deepseek`/`ollama`/`huggingface`/`nomic`（10 家） | Chat/Embeddings/LLM/Rerank 适配，三种继承模式 | [partner-model-providers.md](partner-model-providers/partner-model-providers.md) |
| partner-search-tools | Exa 搜索工具 | `exa/`（17 py） | ExaSearchResults / ExaFindSimilarResults / ExaSearchRetriever | [partner-search-tools.md](partner-search-tools/partner-search-tools.md) |

## 域级机制细节

### 通用集成模式（所有 partner 共享）

1. **基类继承（依赖倒置）**：所有 partner 类继承 `langchain_core` 的抽象基类——
   - Chat 模型 → `BaseChatModel`（实现 `_generate`/`_stream`/`_agenerate`/`_astream`）
   - 嵌入 → `Embeddings`（实现 `embed_documents`/`embed_query`）
   - 文本补全 → `BaseLLM`/`LLM`
   - 向量存储 → `VectorStore`（实现 `add_texts`/`similarity_search`/`from_texts`）
   - 工具 → `BaseTool`（实现 `_run`）
   - 检索器 → `BaseRetriever`（实现 `_get_relevant_documents`）
   - 中间件 → `AgentMiddleware`（`before_model`/`after_model` 钩子）

2. **消息转换层**：每个 Chat 模型 partner 都实现双向转换——
   - langchain `BaseMessage`（HumanMessage/AIMessage/SystemMessage/ToolMessage/FunctionMessage）→ 各 API 原生请求格式；
   - 各 API 响应 → langchain `BaseMessage`（含 `tool_calls`、`additional_kwargs`、`usage_metadata`）。
   - 这是 partner 集成的**核心适配层**，也是各家差异最大的地方。

3. **工具调用转换**：langchain `ToolCall` ↔ 各 API 的 function calling / tool use 格式（OpenAI `tool_calls`、Anthropic `tool_use` block、Groq/Mistral 各自格式）。

4. **错误映射**：各 SDK 异常 → langchain_core 的 `Model*Error` 异常体系（认证/限流/上下文溢出/模型不存在/超时/连接错误），使上层代码可统一捕获。

5. **可选依赖与懒初始化**：每个包在 `validate_environment`（pydantic model_validator）中检测对应 SDK 是否安装、解析 API key（环境变量 → 显式参数 → callable），构建 SDK 客户端。

6. **配置驱动**：模型参数（`model`/`temperature`/`max_tokens` 等）通过 pydantic 字段配置；部分包支持运行时按配置分派到不同 API（如 OpenAI 的 Chat Completions vs Responses API）。

### 三种 Chat 模型继承模式

| 模式 | 代表包 | 说明 |
|---|---|---|
| 直接继承 `BaseChatModel` | ollama、groq、mistralai、perplexity、fireworks、xai、openrouter | 自实现消息转换与错误映射 |
| 继承 `BaseChatOpenAI` | deepseek | 复用 OpenAI 适配层，仅覆盖 endpoint 与 API key（OpenAI 兼容 API 提供商的最简集成） |
| 本地推理 | ollama、huggingface-pipeline | 不调云端 API，在本地进程内运行模型 |

### 17 个集成包规模与分类

| 分类 | 包 | 规模 |
|---|---|---|
| Chat 模型（核心） | openai、anthropic | 最大（84/41 py），功能最全 |
| 向量存储 | qdrant、chroma | 38/14 py |
| 其他模型提供商 | perplexity、fireworks、groq、mistralai、openrouter、xai、deepseek、ollama、huggingface、nomic | 10 家，每家 13-32 py |
| 搜索工具 | exa | 17 py |

### 外部边界

所有第三方 API 服务端（OpenAI/Anthropic/Groq/DeepSeek 等云端）、向量数据库服务端（Qdrant/Chroma）、搜索 API（Exa）、各 Python SDK（openai/anthropic/qdrant-client/chromadb/exa-py 等）、本地推理运行时（ollama 服务/transformers）一律标注「不在本仓库源码内」。本域仅提供适配层。

## 图表清单

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| OpenAI 架构 | partner-openai/partner-openai-architecture.html | architecture | standard |
| OpenAI 时序 | partner-openai/partner-openai-sequence.html | sequence | standard |
| Anthropic 架构 | partner-anthropic/partner-anthropic-architecture.html | architecture | standard |
| Anthropic 时序 | partner-anthropic/partner-anthropic-sequence.html | sequence | standard |
| 向量存储架构 | partner-vector-stores/partner-vector-stores-architecture.html | architecture | standard |
| 向量存储数据流 | partner-vector-stores/partner-vector-stores-dataflow.html | dataflow | standard |
| 模型提供商架构 | partner-model-providers/partner-model-providers-architecture.html | architecture | standard |
| Exa 架构 | partner-search-tools/partner-search-tools-architecture.html | architecture | standard |

所有图均为 standard 档（跨层组件较多，showcase 布局校验未一次通过），各叶子 MD 第 10 节已披露降档原因。
