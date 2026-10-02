# LangChain Monorepo 架构分析文档（第三轮改进版）

> 对 LangChain Python monorepo（branch master，commit `89252a8f7043a74f2300729fd038df8221a87e1e`，2026-09-21）的第三轮深度源码架构分析，按「系统级 → 域 → 叶子」三级拆分，**7 域 35 叶**，产出设计文档级 MD 与 archify 交互式 HTML 图。
>
> **本目录为改进对比版**：基线（第一轮）产物在 `../architecture/`（只读未改动，41 MD / 46 HTML / 46 JSON，7 域 32 叶），本轮改进点清单见 [improve-comparison.md](improve-comparison.md)。本轮按 `diagram-policy.md` 五种图"核心语义"策略出图：系统级 4 张、每域 ≥3 张、每叶 ≥2 张；partners 域由 5 叶深化为 8 叶（17 个集成包逐包归组）。

## 系统级文档

| 文档 | 说明 |
|------|------|
| [系统级总览](system-overview.md) | 功能、解决的问题、系统边界、四种语义模型说明 |
| [系统架构图](system-architecture.html) | Architecture：Monorepo 包分层与依赖（静态拓扑） |
| [代理执行时序图](system-sequence.html) | Sequence：用户→Agent→中间件→模型→工具 调用链 |
| [核心数据流图](system-dataflow.html) | DataFlow：RAG 管道（加载→分块→向量→检索→推理） |
| [代理运行生命周期](system-lifecycle.html) | Lifecycle：一次 agent 运行的状态机 |

---

## 域索引

### 1. [core-primitives — langchain-core 核心抽象](core-primitives/core-primitives.md)

langchain-core 是整个 monorepo 的基础契约层（181 py / 7.0 万行），定义所有抽象基类与 Runnable 协议。域级图：[架构](core-primitives/core-primitives-architecture.html) / [数据流](core-primitives/core-primitives-dataflow.html) / [时序](core-primitives/core-primitives-sequence.html)。

| 叶子 | 设计文档 | 图 |
|------|---------|----|
| core-runnables | [MD](core-primitives/core-runnables/core-runnables.md) | [架构](core-primitives/core-runnables/core-runnables-architecture.html) / [时序](core-primitives/core-runnables/core-runnables-sequence.html) |
| core-messages | [MD](core-primitives/core-messages/core-messages.md) | [架构](core-primitives/core-messages/core-messages-architecture.html) / [数据流](core-primitives/core-messages/core-messages-dataflow.html) |
| core-prompts | [MD](core-primitives/core-prompts/core-prompts.md) | [架构](core-primitives/core-prompts/core-prompts-architecture.html) / [数据流](core-primitives/core-prompts/core-prompts-dataflow.html) |
| core-output-parsers | [MD](core-primitives/core-output-parsers/core-output-parsers.md) | [架构](core-primitives/core-output-parsers/core-output-parsers-architecture.html) / [数据流](core-primitives/core-output-parsers/core-output-parsers-dataflow.html) |
| core-tools | [MD](core-primitives/core-tools/core-tools.md) | [架构](core-primitives/core-tools/core-tools-architecture.html) / [时序](core-primitives/core-tools/core-tools-sequence.html) |
| core-callbacks-tracers | [MD](core-primitives/core-callbacks-tracers/core-callbacks-tracers.md) | [架构](core-primitives/core-callbacks-tracers/core-callbacks-tracers-architecture.html) / [数据流](core-primitives/core-callbacks-tracers/core-callbacks-tracers-dataflow.html) |
| core-language-models | [MD](core-primitives/core-language-models/core-language-models.md) | [架构](core-primitives/core-language-models/core-language-models-architecture.html) / [时序](core-primitives/core-language-models/core-language-models-sequence.html) |
| core-vectorstores-retrievers | [MD](core-primitives/core-vectorstores-retrievers/core-vectorstores-retrievers.md) | [架构](core-primitives/core-vectorstores-retrievers/core-vectorstores-retrievers-architecture.html) / [数据流](core-primitives/core-vectorstores-retrievers/core-vectorstores-retrievers-dataflow.html) |
| core-documents-loaders | [MD](core-primitives/core-documents-loaders/core-documents-loaders.md) | [架构](core-primitives/core-documents-loaders/core-documents-loaders-architecture.html) / [数据流](core-primitives/core-documents-loaders/core-documents-loaders-dataflow.html) |
| core-serialization-cache-infra | [MD](core-primitives/core-serialization-cache-infra/core-serialization-cache-infra.md) | [架构](core-primitives/core-serialization-cache-infra/core-serialization-cache-infra-architecture.html) / [数据流](core-primitives/core-serialization-cache-infra/core-serialization-cache-infra-dataflow.html) |

### 2. [agents — langchain v1 代理编排](agents/agents.md)

langchain v1（40 py / 1.6 万行）当前活跃维护，核心是 `create_agent` 工厂 + 中间件链。域级图：[架构](agents/agents-architecture.html) / [构建执行流程](agents/agents-workflow.html) / [洋葱链时序](agents/agents-onion-sequence.html)。

| 叶子 | 设计文档 | 图 |
|------|---------|----|
| agent-factory | [MD](agents/agent-factory/agent-factory.md) | [架构](agents/agent-factory/agent-factory-architecture.html) / [生命周期](agents/agent-factory/agent-factory-lifecycle.html) |
| agent-middleware | [MD](agents/agent-middleware/agent-middleware.md) | [架构](agents/agent-middleware/agent-middleware-architecture.html) / [数据流](agents/agent-middleware/agent-middleware-dataflow.html) |
| agent-execution-tools | [MD](agents/agent-execution-tools/agent-execution-tools.md) | [架构](agents/agent-execution-tools/agent-execution-tools-architecture.html) / [数据流](agents/agent-execution-tools/agent-execution-tools-dataflow.html) |
| mcp-integration | [MD](agents/mcp-integration/mcp-integration.md) | [架构](agents/mcp-integration/mcp-integration-architecture.html) / [时序](agents/mcp-integration/mcp-integration-sequence.html) |
| chat-models-embeddings | [MD](agents/chat-models-embeddings/chat-models-embeddings.md) | [架构](agents/chat-models-embeddings/chat-models-embeddings-architecture.html) / [时序](agents/chat-models-embeddings/chat-models-embeddings-sequence.html) |
| messages-rate-limiters | [MD](agents/messages-rate-limiters/messages-rate-limiters.md) | [架构](agents/messages-rate-limiters/messages-rate-limiters-architecture.html) / [数据流](agents/messages-rate-limiters/messages-rate-limiters-dataflow.html) |

### 3. [classic — langchain-classic 遗留兼容层](classic/classic.md)

langchain-classic（1321 py / 7.1 万行，包名 `langchain_classic`）遗留包，经典 Chain/Agent 范式。域级图：[架构](classic/classic-architecture.html) / [数据流](classic/classic-dataflow.html) / [时序](classic/classic-sequence.html)。

| 叶子 | 设计文档 | 图 |
|------|---------|----|
| classic-chains | [MD](classic/classic-chains/classic-chains.md) | [架构](classic/classic-chains/classic-chains-architecture.html) / [时序](classic/classic-chains/classic-chains-sequence.html) |
| classic-agents | [MD](classic/classic-agents/classic-agents.md) | [架构](classic/classic-agents/classic-agents-architecture.html) / [生命周期](classic/classic-agents/classic-agents-lifecycle.html) |
| classic-llms-chat | [MD](classic/classic-llms-chat/classic-llms-chat.md) | [架构](classic/classic-llms-chat/classic-llms-chat-architecture.html) / [时序](classic/classic-llms-chat/classic-llms-chat-delegation-sequence.html) |
| classic-memory | [MD](classic/classic-memory/classic-memory.md) | [架构](classic/classic-memory/classic-memory-architecture.html) / [时序](classic/classic-memory/classic-memory-conversation-sequence.html) |
| classic-loaders | [MD](classic/classic-loaders/classic-loaders.md) | [架构](classic/classic-loaders/classic-loaders-architecture.html) / [数据流](classic/classic-loaders/classic-loaders-dataflow.html) |
| classic-retrievers-stores | [MD](classic/classic-retrievers-stores/classic-retrievers-stores.md) | [架构](classic/classic-retrievers-stores/classic-retrievers-stores-architecture.html) / [数据流](classic/classic-retrievers-stores/classic-retrievers-stores-retrieval-dataflow.html) |
| classic-utils-eval | [MD](classic/classic-utils-eval/classic-utils-eval.md) | [架构](classic/classic-utils-eval/classic-utils-eval-architecture.html) / [时序](classic/classic-utils-eval/classic-utils-eval-retry-sequence.html) |
| classic-callbacks-infra | [MD](classic/classic-callbacks-infra/classic-callbacks-infra.md) | [架构](classic/classic-callbacks-infra/classic-callbacks-infra-architecture.html) / [数据流](classic/classic-callbacks-infra/classic-callbacks-infra-event-dispatch-dataflow.html) |

### 4. [partners — 第三方集成](partners/partners.md)

17 个第三方集成包（160 py / 5.7 万行），实现 core 定义的抽象基类。**本轮深化：5 叶 → 8 叶，逐包归组独立成篇**。域级图：[架构](partners/partners-architecture.html) / [时序](partners/partners-sequence.html) / [数据流](partners/partners-dataflow.html)。

| 叶子 | 设计文档 | 图 |
|------|---------|----|
| partner-openai | [MD](partners/partner-openai/partner-openai.md) | [架构](partners/partner-openai/partner-openai-architecture.html) / [时序](partners/partner-openai/partner-openai-sequence.html) |
| partner-anthropic | [MD](partners/partner-anthropic/partner-anthropic.md) | [架构](partners/partner-anthropic/partner-anthropic-architecture.html) / [时序](partners/partner-anthropic/partner-anthropic-sequence.html) |
| partner-qdrant | [MD](partners/partner-qdrant/partner-qdrant.md) | [架构](partners/partner-qdrant/partner-qdrant-architecture.html) / [数据流](partners/partner-qdrant/partner-qdrant-dataflow.html) |
| partner-chroma | [MD](partners/partner-chroma/partner-chroma.md) | [架构](partners/partner-chroma/partner-chroma-architecture.html) / [时序](partners/partner-chroma/partner-chroma-sequence.html) |
| partner-llm-api-providers | [MD](partners/partner-llm-api-providers/partner-llm-api-providers.md) | [架构](partners/partner-llm-api-providers/partner-llm-api-providers-architecture.html) / [时序](partners/partner-llm-api-providers/partner-llm-api-providers-sequence.html) |
| partner-local-inference | [MD](partners/partner-local-inference/partner-local-inference.md) | [架构](partners/partner-local-inference/partner-local-inference-architecture.html) / [数据流](partners/partner-local-inference/partner-local-inference-dataflow.html) |
| partner-nomic | [MD](partners/partner-nomic/partner-nomic.md) | [架构](partners/partner-nomic/partner-nomic-architecture.html) / [数据流](partners/partner-nomic/partner-nomic-dataflow.html) |
| partner-search-tools | [MD](partners/partner-search-tools/partner-search-tools.md) | [架构](partners/partner-search-tools/partner-search-tools-architecture.html) / [时序](partners/partner-search-tools/partner-search-tools-sequence.html) |

### 5. [text-splitters — 文档分块工具](text-splitters/text-splitters.md)

独立包（23 py / 8.7k 行）。域级图：[域架构](text-splitters/text-splitters-domain-architecture.html)；叶子图：[架构](text-splitters/text-splitters/text-splitters-architecture.html) / [数据流](text-splitters/text-splitters/text-splitters-dataflow.html)。

### 6. [testing-infra — 标准化测试套件](testing-infra/testing-infra.md)

共享集成测试套件（36 py / 1.1 万行）。域级图：[域架构](testing-infra/testing-infra-domain-architecture.html)；叶子图：[架构](testing-infra/standard-tests/standard-tests-architecture.html) / [接入流程](testing-infra/standard-tests/standard-tests-workflow.html)。

### 7. [model-profiles — 模型能力配置](model-profiles/model-profiles.md)

CLI 工具（9 py / 2k 行）。域级图：[域架构](model-profiles/model-profiles-domain-architecture.html)；叶子图：[架构](model-profiles/model-profiles/model-profiles-architecture.html) / [数据流](model-profiles/model-profiles/model-profiles-dataflow.html)。

---

## 产出统计

| 层级 | MD | HTML | JSON |
|------|----|------|------|
| 系统级（README + 总览 + 4 图） | 2 | 4 | 4 |
| 域级（7 域总览 + 每域 ≥3 图） | 7 | 15 | 15 |
| 叶子级（35 叶 × 1 MD + 2 图） | 35 | 70 | 70 |
| 对比说明 | 1 | — | — |
| **合计** | **45** | **89** | **89** |

## 分析说明

- **语言适配口径**：纯 Python 项目，按「能力缝 + 注册表/工厂 + 配置驱动 + 可选依赖懒加载」范式分析（非 controller/reconciler 模式），已写入各叶子第 9 节。
- **图型选择**：每张图先按 diagram-policy §1–§5 确认核心语义（静态拓扑/泳道流程/消息时序/数据变换/单实体状态机五选一），不匹配不画，省略原因写入对应 MD 第 10 节。
- **质量档位**：本轮 89 张图中 showcase 通过率 **77.5%**（回归对比第二轮基线 61.4%，+16.1 个百分点，0 退步）；落 standard 的图均在各叶 MD 第 10 节披露失败检查名与修复动作。
- **外部边界**：OpenAI/Anthropic/各模型提供商 API、向量数据库服务端、MCP 服务器、LangSmith、各 Python SDK、本地推理运行时等均标注「不在本仓库源码内」。
- **本轮变化**：partners 域深化至 8 叶（qdrant/chroma 分叶、llm-api-providers/local-inference/nomic 从 model-providers 拆出）；全部叶子关键调用链行号按当前 HEAD 实读核验；上游 archify CLI 升级后统一补齐 `meta.output` schema 字段并重渲染。
- **源码基准**：与第二轮（4492ad7a）之间仅 1 个 docs 部署 commit（89252a8f），无源码改动；git 改动严格限定于本目录，基线 `docs/architecture/` 未触碰。
