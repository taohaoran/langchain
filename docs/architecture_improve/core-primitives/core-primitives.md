# 原语层（core-primitives）域总览

> 本域是 langchain-core 的 10 个基础原语叶子。各叶子详情见对应文档。
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/`。

## 1. 域职责

`core-primitives` 域是 LangChain 的**依赖倒置基础抽象层**：以抽象基类（ABC）定义契约，实现方在 `libs/partners/*`（OpenAI/Anthropic 等）与外部向量库。域内所有原语都实现 `Runnable` 接口，因此可经 LCEL（`|` 管道符）自由组合成端到端链。核心代码路径：`runnables/base.py`（组合引擎）→ 各 `*_base.py`（原语契约）→ partners 实现。

本域不承载实际计算：模型推理在外部 API、向量检索在外部向量库、token 计算在第三方库——仓库价值 = 统一接口 + Runnable 组合 + 可插拔钩子（回调/缓存/序列化）。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序图 / 数据流图 | 职责一句话 |
|------|------|--------|-------------------|-----------|
| 组合引擎 Runnable | [core-runnables.md](core-runnables/core-runnables.md) | [架构图](core-runnables/core-runnables-architecture.html) | [时序图](core-runnables/core-runnables-sequence.html) | 所有原语的组合引擎，LCEL `|` 与 invoke/batch/stream |
| 消息 | [core-messages.md](core-messages/core-messages.md) | [架构图](core-messages/core-messages-architecture.html) | [数据流](core-messages/core-messages-dataflow.html) | BaseMessage 体系与多 provider 格式归一化 |
| 提示词 | [core-prompts.md](core-prompts/core-prompts.md) | [架构图](core-prompts/core-prompts-architecture.html) | [数据流](core-prompts/core-prompts-dataflow.html) | PromptTemplate/ChatPromptTemplate 与 PromptValue |
| 输出解析 | [core-output-parsers.md](core-output-parsers/core-output-parsers.md) | [架构图](core-output-parsers/core-output-parsers-architecture.html) | [数据流](core-output-parsers/core-output-parsers-dataflow.html) | 把模型文本输出解析成结构化对象 |
| 工具 | [core-tools.md](core-tools/core-tools.md) | [架构图](core-tools/core-tools-architecture.html) | [时序图](core-tools/core-tools-sequence.html) | 供模型调用的函数/工具抽象与调用管道 |
| 回调与追踪 | [core-callbacks-tracers.md](core-callbacks-tracers/core-callbacks-tracers.md) | [架构图](core-callbacks-tracers/core-callbacks-tracers-architecture.html) | [数据流](core-callbacks-tracers/core-callbacks-tracers-dataflow.html) | 横切追踪/日志/流式的回调注入点 |
| 语言模型 | [core-language-models.md](core-language-models/core-language-models.md) | [架构图](core-language-models/core-language-models-architecture.html) | [时序图](core-language-models/core-language-models-sequence.html) | 聊天/补全/交叉编码器抽象基类 |
| 向量库与检索器 | [core-vectorstores-retrievers.md](core-vectorstores-retrievers/core-vectorstores-retrievers.md) | [架构图](core-vectorstores-retrievers/core-vectorstores-retrievers-architecture.html) | [数据流](core-vectorstores-retrievers/core-vectorstores-retrievers-dataflow.html) | 向量存储/嵌入/检索器与增量索引契约 |
| 文档与加载器 | [core-documents-loaders.md](core-documents-loaders/core-documents-loaders.md) | [架构图](core-documents-loaders/core-documents-loaders-architecture.html) | [数据流](core-documents-loaders/core-documents-loaders-dataflow.html) | Document/Blob 数据结构与加载/转换契约 |
| 序列化缓存基础设施 | [core-serialization-cache-infra.md](core-serialization-cache-infra/core-serialization-cache-infra.md) | [架构图](core-serialization-cache-infra/core-serialization-cache-infra-architecture.html) | [数据流](core-serialization-cache-infra/core-serialization-cache-infra-dataflow.html) | LC 序列化/LLM 缓存/KV/限流/异常/安全策略 |

## 3. 域级机制细节

- **Runnable 即一切**：每个原语都是 `RunnableSerializable` 子类，因此 `prompt | model | parser` 这种组合是统一机制，而非各原语各自实现。`Runnable.invoke` 经 `_call_with_config` 统一注入回调管理器、处理 contextvars 与流式（见 `core-runnables`）。
- **依赖倒置**：契约在消费方基类（本域），实现在 partners（外部）。本域用「注册表/工厂」做字符串路由：`search_type` 选检索策略、`@tool` 注册工具、`as_retriever` 做 VectorStore→Retriever 转换。
- **横切钩子**：回调（`core-callbacks-tracers`）、缓存（`BaseCache`）、序列化（`Serializable`）是贯穿所有原语的三条横切缝，由基础设施叶子统一提供。
- **外部边界**：Partners API、向量数据库、tokenizer、LangSmith 云均不在本仓库源码内，各叶子第 7 节已逐一声明。

## 4. 域级图

![core-primitives 域组件拓扑](core-primitives-architecture.html)

![原语组成 RAG 链数据流](core-primitives-dataflow.html)

![Runnable.invoke 跨原语调用时序](core-primitives-sequence.html)

三张域级图分别对应三种语义模型：**architecture**（10 个原语的静态组件拓扑与外部边界）、**dataflow**（query→Retriever→Prompt→ChatModel→OutputParser 的数据流动，即原语如何组成 RAG 链）、**sequence**（一次 invoke 在 Runnable 引擎下跨各原语子步的消息交互时序）。三者语义不重复：空间结构 / 数据流向 / 时间顺序互补。

本域未生成 lifecycle 图：域内无单实体多状态机（Runnable 的 run 状态已由回调/Run 树表达，属多参与方交互而非单一实体状态变迁），且与已有 sequence 信息重复，按资源节省原则省略。
