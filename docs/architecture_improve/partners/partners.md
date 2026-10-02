# partners 域总览

> 本文是 `partners` 域的总览文档。本域归并 LangChain monorepo 中 `libs/partners/` 下的全部第三方集成包分析。
> 系统级总览与 README 索引导航由汇总者统一产出（位于本目录上级）。
> 本目录为第三轮（improve）分析产物，基线只读见 `../../architecture/partners/`（不得修改）。
>
> 源码基准：`libs/partners/`，branch `master`，commit `89252a8f`。

## 1. 域职责

`partners` 域是 LangChain 的**集成适配层**：把第三方 LLM 提供商、向量数据库、搜索 API 等外部系统适配到 `langchain_core` 定义的统一抽象接口上。本域不承载实际计算——所有推理、向量检索、搜索均发生在外部服务端或用户注入的组件中；本仓库价值在于**统一接口 + 消息/数据格式转换 + 错误映射 + 懒初始化**。

本域覆盖 `libs/partners/` 下 17 个集成包，第三轮按功能细分为 8 个叶子子系统。

## 2. 叶子索引表

| 叶子 | 文档 | 覆盖包 | 核心能力 | 图 |
|---|---|---|---|---|
| partner-openai | [partner-openai.md](partner-openai/partner-openai.md) | `openai/`（26 py） | ChatOpenAI / AzureChatOpenAI / OpenAIEmbeddings / 工具 / 中间件 | [架构](partner-openai/partner-openai-architecture.html) · [时序](partner-openai/partner-openai-sequence.html) |
| partner-anthropic | [partner-anthropic.md](partner-anthropic/partner-anthropic.md) | `anthropic/`（18 py） | ChatAnthropic / 工具中间件族 / Prompt 缓存 | [架构](partner-anthropic/partner-anthropic-architecture.html) · [时序](partner-anthropic/partner-anthropic-sequence.html) |
| partner-qdrant | [partner-qdrant.md](partner-qdrant/partner-qdrant.md) | `qdrant/`（9 py） | QdrantVectorStore 稠密+稀疏混合检索 | [架构](partner-qdrant/partner-qdrant-architecture.html) · [数据流](partner-qdrant/partner-qdrant-dataflow.html) |
| partner-chroma | [partner-chroma.md](partner-chroma/partner-chroma.md) | `chroma/`（5 py） | Chroma 文本+图像+混合检索 | [架构](partner-chroma/partner-chroma-architecture.html) · [时序](partner-chroma/partner-chroma-sequence.html) |
| partner-llm-api-providers | [partner-llm-api-providers.md](partner-llm-api-providers/partner-llm-api-providers.md) | `perplexity`/`fireworks`/`groq`/`mistralai`/`openrouter`/`xai`/`deepseek`（7 家云端） | 两种继承模式：复用 BaseChatOpenAI vs 自实现 | [架构](partner-llm-api-providers/partner-llm-api-providers-architecture.html) · [时序](partner-llm-api-providers/partner-llm-api-providers-sequence.html) |
| partner-local-inference | [partner-local-inference.md](partner-local-inference/partner-local-inference.md) | `ollama/` + `huggingface/`（本地进程内推理） | ChatOllama / HF Pipeline / HF Endpoint | [架构](partner-local-inference/partner-local-inference-architecture.html) · [数据流](partner-local-inference/partner-local-inference-dataflow.html) |
| partner-nomic | [partner-nomic.md](partner-nomic/partner-nomic.md) | `nomic/`（5 py） | NomicEmbeddings 文本+图像嵌入 | [架构](partner-nomic/partner-nomic-architecture.html) · [数据流](partner-nomic/partner-nomic-dataflow.html) |
| partner-search-tools | [partner-search-tools.md](partner-search-tools/partner-search-tools.md) | `exa/`（7 py） | ExaSearchResults / Retriever | [架构](partner-search-tools/partner-search-tools-architecture.html) · [时序](partner-search-tools/partner-search-tools-sequence.html) |

## 3. 域级机制细节

### 通用集成模式（所有 partner 共享）

1. **基类继承（依赖倒置）**：所有 partner 类继承 `langchain_core` 的抽象基类——Chat 模型 → `BaseChatModel`；嵌入 → `Embeddings`；文本补全 → `BaseLLM`/`LLM`；向量存储 → `VectorStore`；工具 → `BaseTool`；检索器 → `BaseRetriever`；中间件 → `AgentMiddleware`。
2. **消息转换层**：每个 Chat partner 实现双向转换——langchain `BaseMessage` ↔ 各 API 原生格式。这是核心适配层，也是各家差异最大处。
3. **工具调用转换**：langchain `ToolCall` ↔ 各 API 的 function calling / tool use 格式。
4. **错误映射**：各 SDK 异常 → langchain_core `Model*Error` 异常体系。
5. **可选依赖与懒初始化**：`validate_environment` 检测 SDK、解析 API key、构建客户端。
6. **配置驱动**：pydantic 字段配置；部分包按配置分派到不同 API（OpenAI Responses API、Anthropic betas/thinking、Qdrant RetrievalMode）。

### Chat 模型继承模式（第三轮细化）

| 模式 | 代表包 | 说明 |
|---|---|---|
| 直接继承 `BaseChatModel` | groq、mistralai、openrouter、perplexity、fireworks、ollama、huggingface | 自实现消息转换与错误映射 |
| 继承 `BaseChatOpenAI` | deepseek、xai | 复用 OpenAI 适配层，仅覆盖 endpoint 与 API key |
| 本地进程内推理 | ollama（守护进程）、huggingface-pipeline（transformers） | 不调云端 API |

> **事实修正**：`ChatOpenRouter` 继承 `BaseChatModel`（自实现），不是 `BaseChatOpenAI`。

### 外部边界

所有第三方 API 服务端、向量数据库服务端、搜索 API、各 Python SDK、本地推理运行时一律标注「不在本仓库源码内」。本域仅提供适配层。

## 4. 域级图

![partners 域集成适配架构图](partners-architecture.html)

![partners 域 Chat 请求通用调用链时序](partners-sequence.html)

![partners 域请求消息变换数据流](partners-dataflow.html)

## 5. 域级图表清单与质量

| 图 | 文件 | 类型 | 质量档 |
|---|---|---|---|
| partners 域集成适配架构图 | `partners-architecture.html` | architecture | standard |
| partners 域 Chat 请求通用调用链时序 | `partners-sequence.html` | sequence | **showcase** |
| partners 域请求消息变换数据流 | `partners-dataflow.html` | dataflow | **showcase** |

三张域级图对应三种不同核心语义：architecture 表达静态组件拓扑，sequence 表达单次请求的时间序消息交互，dataflow 表达数据载体的变换管道。

**第三轮变更**：叶子由 5 叶细分为 8 叶（原 partner-vector-stores 拆为 partner-qdrant + partner-chroma；原 partner-model-providers 拆为 partner-llm-api-providers + partner-local-inference + partner-nomic）。域级图补全 `meta.output` 字段后重渲染，档位与第二轮一致。叶子级图本轮全部达到 showcase 或 standard 披露标准。

## 6. 覆盖范围与缺口

- 深读：openai、anthropic、qdrant、chroma、exa、deepseek、xai、groq、openrouter、ollama 关键入口已逐行核验行号。
- 共性列表（未逐行深读）：mistralai/perplexity/fireworks 的 `_generate` 内部消息转换与 groq 同构，以共性说明。
- 所有事实来自 `libs/partners/` 源码（HEAD `89252a8f`），第三方服务端与 SDK 均标注"不在本仓库源码内"。
