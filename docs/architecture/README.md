# LangChain Monorepo 架构分析文档

> 对 LangChain Python monorepo（branch master，commit 4492ad7）执行的深度源码架构分析，按「系统级 → 域 → 叶子子系统」三级拆分，共 **7 个域、32 个叶子子系统**，产出设计文档级 MD 与 archify 交互式 HTML 图。

## 系统级文档

| 文档 | 说明 |
|------|------|
| [系统级总览](system-overview.md) | 功能总览、解决的问题、系统边界、架构说明 |
| [系统架构图](system-architecture.html) | Monorepo 包分层与依赖关系（交互式） |
| [代理执行时序图](system-sequence.html) | Agent 从请求到响应的完整调用链（交互式） |
| [核心数据流图](system-dataflow.html) | RAG 管道：文档加载→分块→向量化→检索→推理→追踪（交互式） |

---

## 域索引

### 1. [core-primitives — langchain-core 核心抽象](core-primitives/core-primitives.md)

langchain-core 是整个 monorepo 的基础契约层（181 py / 7.0 万行），定义所有抽象基类与 Runnable 协议，被 v1、classic、partners 共同依赖。

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| core-runnables | [MD](core-primitives/core-runnables/core-runnables.md) | [架构图](core-primitives/core-runnables/core-runnables-architecture.html) / [时序图](core-primitives/core-runnables/core-runnables-sequence.html) |
| core-messages | [MD](core-primitives/core-messages/core-messages.md) | [架构图](core-primitives/core-messages/core-messages-architecture.html) |
| core-prompts | [MD](core-primitives/core-prompts/core-prompts.md) | [架构图](core-primitives/core-prompts/core-prompts-architecture.html) |
| core-output-parsers | [MD](core-primitives/core-output-parsers/core-output-parsers.md) | [架构图](core-primitives/core-output-parsers/core-output-parsers-architecture.html) |
| core-tools | [MD](core-primitives/core-tools/core-tools.md) | [架构图](core-primitives/core-tools/core-tools-architecture.html) |
| core-callbacks-tracers | [MD](core-primitives/core-callbacks-tracers/core-callbacks-tracers.md) | [数据流图](core-primitives/core-callbacks-tracers/core-callbacks-tracers-dataflow.html) |
| core-language-models | [MD](core-primitives/core-language-models/core-language-models.md) | [架构图](core-primitives/core-language-models/core-language-models-architecture.html) |
| core-vectorstores-retrievers | [MD](core-primitives/core-vectorstores-retrievers/core-vectorstores-retrievers.md) | [架构图](core-primitives/core-vectorstores-retrievers/core-vectorstores-retrievers-architecture.html) |
| core-documents-loaders | [MD](core-primitives/core-documents-loaders/core-documents-loaders.md) | [架构图](core-primitives/core-documents-loaders/core-documents-loaders-architecture.html) |
| core-serialization-cache-infra | [MD](core-primitives/core-serialization-cache-infra/core-serialization-cache-infra.md) | [架构图](core-primitives/core-serialization-cache-infra/core-serialization-cache-infra-architecture.html) |

### 2. [agents — langchain v1 代理编排](agents/agents.md)

langchain v1（libs/langchain_v1，40 py / 1.6 万行）是当前活跃维护的包，核心是 `create_agent` 工厂 + 中间件链的代理编排范式。

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| agent-factory | [MD](agents/agent-factory/agent-factory.md) | [架构图](agents/agent-factory/agent-factory-architecture.html) / [生命周期图](agents/agent-factory/agent-factory-lifecycle.html) |
| agent-middleware | [MD](agents/agent-middleware/agent-middleware.md) | [架构图](agents/agent-middleware/agent-middleware-architecture.html) / [数据流图](agents/agent-middleware/agent-middleware-dataflow.html) |
| agent-execution-tools | [MD](agents/agent-execution-tools/agent-execution-tools.md) | [架构图](agents/agent-execution-tools/agent-execution-tools-architecture.html) |
| mcp-integration | [MD](agents/mcp-integration/mcp-integration.md) | [架构图](agents/mcp-integration/mcp-integration-architecture.html) / [时序图](agents/mcp-integration/mcp-integration-sequence.html) |
| chat-models-embeddings | [MD](agents/chat-models-embeddings/chat-models-embeddings.md) | [架构图](agents/chat-models-embeddings/chat-models-embeddings-architecture.html) |
| messages-rate-limiters | [MD](agents/messages-rate-limiters/messages-rate-limiters.md) | [架构图](agents/messages-rate-limiters/messages-rate-limiters-architecture.html) |

### 3. [classic — langchain-classic 遗留兼容层](classic/classic.md)

langchain-classic（libs/langchain，1321 py / 7.1 万行，包名 langchain_classic）是遗留包（"no new features"），包含经典 Chain/Agent 范式与大量兼容转发层。

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| classic-chains | [MD](classic/classic-chains/classic-chains.md) | [架构图](classic/classic-chains/classic-chains-architecture.html) / [时序图](classic/classic-chains/classic-chains-sequence.html) |
| classic-agents | [MD](classic/classic-agents/classic-agents.md) | [架构图](classic/classic-agents/classic-agents-architecture.html) / [生命周期图](classic/classic-agents/classic-agents-lifecycle.html) |
| classic-llms-chat | [MD](classic/classic-llms-chat/classic-llms-chat.md) | [架构图](classic/classic-llms-chat/classic-llms-chat-architecture.html) |
| classic-memory | [MD](classic/classic-memory/classic-memory.md) | [架构图](classic/classic-memory/classic-memory-architecture.html) |
| classic-loaders | [MD](classic/classic-loaders/classic-loaders.md) | [数据流图](classic/classic-loaders/classic-loaders-dataflow.html) |
| classic-retrievers-stores | [MD](classic/classic-retrievers-stores/classic-retrievers-stores.md) | [架构图](classic/classic-retrievers-stores/classic-retrievers-stores-architecture.html) |
| classic-utils-eval | [MD](classic/classic-utils-eval/classic-utils-eval.md) | [架构图](classic/classic-utils-eval/classic-utils-eval-architecture.html) |
| classic-callbacks-infra | [MD](classic/classic-callbacks-infra/classic-callbacks-infra.md) | [架构图](classic/classic-callbacks-infra/classic-callbacks-infra-architecture.html) |

### 4. [partners — 第三方集成](partners/partners.md)

17 个第三方集成包（160 py / 5.7 万行），实现 langchain-core 定义的抽象基类，是典型的「接口在消费方、实现在集成方」依赖倒置。

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| partner-openai | [MD](partners/partner-openai/partner-openai.md) | [架构图](partners/partner-openai/partner-openai-architecture.html) / [时序图](partners/partner-openai/partner-openai-sequence.html) |
| partner-anthropic | [MD](partners/partner-anthropic/partner-anthropic.md) | [架构图](partners/partner-anthropic/partner-anthropic-architecture.html) / [时序图](partners/partner-anthropic/partner-anthropic-sequence.html) |
| partner-vector-stores | [MD](partners/partner-vector-stores/partner-vector-stores.md) | [架构图](partners/partner-vector-stores/partner-vector-stores-architecture.html) / [数据流图](partners/partner-vector-stores/partner-vector-stores-dataflow.html) |
| partner-model-providers | [MD](partners/partner-model-providers/partner-model-providers.md) | [架构图](partners/partner-model-providers/partner-model-providers-architecture.html) |
| partner-search-tools | [MD](partners/partner-search-tools/partner-search-tools.md) | [架构图](partners/partner-search-tools/partner-search-tools-architecture.html) |

### 5. [text-splitters — 文档分块工具](text-splitters/text-splitters.md)

独立包（23 py / 8.7k 行），提供 RecursiveCharacterTextSplitter 等文档分块能力。

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| text-splitters | [MD](text-splitters/text-splitters/text-splitters.md) | [架构图](text-splitters/text-splitters/text-splitters-architecture.html) / [数据流图](text-splitters/text-splitters/text-splitters-dataflow.html) |

### 6. [testing-infra — 标准化测试套件](testing-infra/testing-infra.md)

共享集成测试套件（36 py / 1.1 万行），为 partners 集成提供标准化测试契约与防覆盖守卫。

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| standard-tests | [MD](testing-infra/standard-tests/standard-tests.md) | [架构图](testing-infra/standard-tests/standard-tests-architecture.html) |

### 7. [model-profiles — 模型能力配置](model-profiles/model-profiles.md)

CLI 工具（9 py / 2k 行），从模型 API 刷新能力标志与上下文窗口等 profile 数据。

| 叶子 | 设计文档 | 架构图 |
|------|---------|--------|
| model-profiles | [MD](model-profiles/model-profiles/model-profiles.md) | [架构图](model-profiles/model-profiles/model-profiles-architecture.html) / [数据流图](model-profiles/model-profiles/model-profiles-dataflow.html) |

---

## 分析说明

- **语言适配口径**：纯 Python 项目，按「能力缝 + 注册表/工厂 + 配置驱动 + 可选依赖懒加载」范式分析，非 controller/reconciler 模式。
- **质量档位**：系统级 3 图与大部分叶子图为 `standard` 档（跨层连接复杂，showcase 布局校验未全过），部分简单叶子图达 `showcase`；降档原因在各叶 MD 第 10 节如实披露。
- **外部边界**：OpenAI/Anthropic API、向量数据库服务端、MCP 服务器、LangSmith 平台等均标注「不在本仓库源码内」。
- **产出统计**：32 叶子 MD + 7 域总览 + 系统级总览/README = 41 份 MD；47 张交互式 HTML 图；47 份 JSON IR。
