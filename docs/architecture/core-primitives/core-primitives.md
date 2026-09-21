# 核心原语（core-primitives）域总览

> 本域包含 langchain-core 包下的 10 个叶子子系统；各叶子详情见对应文档。
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/`（约 181 个 `.py` 文件 / 7.0 万行）。

## 1. 域职责

langchain-core 是整个 LangChain monorepo 的**基础抽象层**：所有上层包（`langchain_v1`、`langchain_classic`）与第三方集成（`libs/partners/*`）都依赖它。本域不承载实际的 LLM 计算或向量数据库实现，而是定义一套**统一的能力契约与组合协议**：

- 以 `Runnable` 抽象把任意一步工作（模型调用、检索、解析、工具）变成可组合、可异步、可流式、可批处理的执行单元；
- 以 `BaseMessage`/`Document`/`BasePromptTemplate`/`BaseOutputParser`/`BaseTool`/`BaseChatModel`/`VectorStore`/`Embeddings` 等抽象基类定义数据结构与行为契约，实现方在 partners（依赖倒置）；
- 以 `callbacks`/`tracers` 作为横切关注点（追踪、日志、流式）的注入点；
- 以 `Serializable`/`BaseCache`/`BaseStore`/`import_attr` 提供序列化、缓存、懒加载等基础设施。

实际的 LLM API 调用、向量数据库、嵌入模型推理都在本仓库之外（partners/外部服务），本域只编排与适配。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/数据流图 | 职责一句话 |
|------|------|--------|---------------|-----------|
| 可组合执行单元 | [core-runnables.md](core-runnables/core-runnables.md) | [架构图](core-runnables/core-runnables-architecture.html) | [时序图](core-runnables/core-runnables-sequence.html) | LCEL 的 Runnable 协议与组合原语（invoke/batch/stream/`|`） |
| 消息类型体系 | [core-messages.md](core-messages/core-messages.md) | [架构图](core-messages/core-messages-architecture.html) | — | BaseMessage 子类与归一化管道（Human/AI/Tool 等） |
| 提示词模板 | [core-prompts.md](core-prompts/core-prompts.md) | [架构图](core-prompts/core-prompts-architecture.html) | — | PromptTemplate/ChatPromptTemplate 与少样本选择器 |
| 输出解析器 | [core-output-parsers.md](core-output-parsers/core-output-parsers.md) | [架构图](core-output-parsers/core-output-parsers-architecture.html) | — | 把模型文本解析成结构化对象（JSON/Pydantic/XML） |
| 工具抽象 | [core-tools.md](core-tools/core-tools.md) | [架构图](core-tools/core-tools-architecture.html) | — | BaseTool 契约与 @tool 装饰器、schema 推断 |
| 回调与追踪 | [core-callbacks-tracers.md](core-callbacks-tracers/core-callbacks-tracers.md) | — | [数据流图](core-callbacks-tracers/core-callbacks-tracers-dataflow.html) | 横切回调注入点与 Run 追踪持久化 |
| 语言模型抽象 | [core-language-models.md](core-language-models/core-language-models.md) | [架构图](core-language-models/core-language-models-architecture.html) | — | BaseChatModel/LLM/BaseCrossEncoder 契约 |
| 向量库与检索器 | [core-vectorstores-retrievers.md](core-vectorstores-retrievers/core-vectorstores-retrievers.md) | [架构图](core-vectorstores-retrievers/core-vectorstores-retrievers-architecture.html) | — | VectorStore/Embeddings/BaseRetriever 与增量索引 |
| 文档与加载器 | [core-documents-loaders.md](core-documents-loaders/core-documents-loaders.md) | [架构图](core-documents-loaders/core-documents-loaders-architecture.html) | — | Document/Blob、BaseLoader、BaseChatMessageHistory |
| 序列化缓存基础设施 | [core-serialization-cache-infra.md](core-serialization-cache-infra/core-serialization-cache-infra.md) | [架构图](core-serialization-cache-infra/core-serialization-cache-infra-architecture.html) | — | Serializable、BaseCache、BaseStore、限流、SSRF、懒加载 |

## 3. 域级机制细节

### 3.1 Runnable 协议贯穿全栈

`Runnable`（`runnables/base.py:133`）是本域的核心能力缝。它定义六种执行原语——`invoke`/`ainvoke`/`batch`/`abatch`/`stream`/`astream`——以及 `transform`/`astream_log`/`astream_events`。所有模型、prompt、parser、tool、retriever 都实现或消费它。`|` 运算符与字典字面量把它们声明式地串成 `RunnableSequence`/`RunnableParallel`，组合出的链自动获得 sync/async/batch/stream 全能力。执行经 `_call_with_config`/`_batch_with_config`/`_transform_stream_with_config` 统一注入回调管理器与配置上下文。

### 3.2 抽象基类定义在消费方（依赖倒置）

`BaseChatModel`、`VectorStore`、`Embeddings`、`BaseTool`、`BaseRetriever`、`BaseChatMessageHistory` 等契约全部定义在 `langchain_core` 内，而具体实现（OpenAI 模型、Chroma 向量库等）在 `libs/partners/*`。上层包与 partners 只依赖本域的抽象，本域不反向依赖它们——这是典型的依赖倒置，也是 langchain-core 能作为稳定基座的原因。

### 3.3 消息类型体系是 LLM 交互通用数据结构

`BaseMessage`（`messages/base.py:93`）的子类（Human/AI/System/Tool/Function/Chat + 各自 Chunk 变体）是 chat model 的输入输出；`type` 字段 + `_create_message_from_message_type` 构成字符串→类的反序列化注册表；`BaseMessageChunk.__add__` 把流式分片合并成完整消息。Prompt 产它、模型消费它、工具结果以 `ToolMessage` 回传它。

### 3.4 callbacks 是横切注入点

每个 Runnable 执行入口经 `get_callback_manager_for_config` 创建 `CallbackManager`，向所有注册的 `BaseCallbackHandler`/`BaseTracer` 广播 `on_*` 事件；`BaseTracer` 把事件聚合成 `Run` 树并持久化（LangSmith 云后端不在本仓库源码内）。run 树经 `get_child` 派生，contextvars 在并发间传播。

### 3.5 懒加载与可选依赖

各包 `__init__.py` 用模块级 `__getattr__` + `_import_utils.import_attr` 实现符号惰性导入，`import langchain_core.runnables` 不立即加载全部重实现；`content_blocks` 等按需延迟 import provider 转换器以规避循环依赖。

### 3.6 安全模型

`load/` 的反序列化走类路径白名单 + jinja2 模板注入拦截；`_security/` 的 `SSRFPolicy`/`validate_safe_url` 防护外网请求，是 LangChain 防 RCE 的核心。

## 4. 质量档位说明

本域共渲染 11 张图（10 叶各 1 张架构图 + runnables 1 张时序图）。因 archify showcase 档对「中心节点多向扇出」「跨层连线」「同 row 连边穿节点」布局约束严格，多数架构图在反复调整布局后仍降为 standard 档；时序图与布局简洁的图（messages/language-models/vectorstores/序列化基础设施）通过 showcase。所有图均实际渲染成功（退出码 0、HTML 非空约 800KB）。降档原因已在各叶子 MD 第 10 节如实披露。
