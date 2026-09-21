# classic 域：langchain_classic 经典包

> 本文是 `docs/architecture/` 下 `classic` 域的总览。系统级总览见 `../system-overview.md`。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `4492ad7a8`，
> 源码位于 `libs/langchain/langchain_classic/`（1321 个 py 文件）。

## 域职责概述

`langchain-classic` 是 LangChain 历史包的**遗留兼容层**。README 明确定位：
"Legacy chains, langchain-community re-exports, indexing API, deprecated functionality"，
并建议"在大多数情况下应使用主 `langchain` 包"。它不承载新功能，作用是让历史代码继续可用。

从架构看，本域有三种角色：

1. **真正仍有实现的控制流**：Chain 抽象（模板方法）、Agent 执行循环、Memory 接口、
   组合型检索器、LLM 评审链——这些是经典范式的核心，但均已标注弃用并引导迁移到
   LCEL / langgraph。
2. **兼容/转发层**：大量 `create_importer` 延迟导入桩，把旧导入路径转发到
   `langchain-core`（基类下沉）与 `langchain-community`/独立 partner 包（实现迁出）。
   llms-chat、loaders、utils-eval、callbacks-infra 四叶主体即此。
3. **经典包特有机制**：`hub.pull/push` 从 LangSmith Hub 拉取序列化对象、`load_chain`/
   `load_agent` 按 `_type` 注册表反序列化。

## 叶子索引表

| # | 叶子 | 职责 | 主要源码 |
| --- | --- | --- | --- |
| 1 | [classic-chains](classic-chains/classic-chains.md) | Chain 抽象与文档合并/路由/工具型链 | `chains/` |
| 2 | [classic-agents](classic-agents/classic-agents.md) | Agent 决策契约与 ReAct 执行循环 | `agents/` |
| 3 | [classic-llms-chat](classic-llms-chat/classic-llms-chat.md) | LLM/ChatModel 基类与厂商转发层 | `llms/`、`chat_models/`、`base_language.py` |
| 4 | [classic-memory](classic-memory/classic-memory.md) | 对话记忆抽象与持久化后端 | `memory/`、`base_memory.py` |
| 5 | [classic-loaders](classic-loaders/classic-loaders.md) | 文档加载器集合与文本分割 | `document_loaders/`、`document_transformers/`、`text_splitter.py` |
| 6 | [classic-retrievers-stores](classic-retrievers-stores/classic-retrievers-stores.md) | 检索器、向量存储、键值存储、索引 | `retrievers/`、`vectorstores/`、`docstore/`、`storage/`、`indexes/` |
| 7 | [classic-utils-eval](classic-utils-eval/classic-utils-eval.md) | 输出解析器、评估框架、工具集转发 | `output_parsers/`、`prompts/`、`evaluation/`、`utilities/` |
| 8 | [classic-callbacks-infra](classic-callbacks-infra/classic-callbacks-infra.md) | 回调、缓存、Hub、序列化基础设施 | `callbacks/`、`cache.py`、`hub.py`、`load/`、`globals.py` |

## 域级机制细节

### 与 langchain-core / langchain v1 的边界

- **基类已下沉 core**：`BaseLanguageModel`、`BaseLLM`、`BaseChatModel`、`BaseLoader`、
  `VectorStore`、`BaseRetriever`、`BaseOutputParser`、`BaseCallbackHandler`、
  `BasePromptTemplate` 的真正定义都在 `langchain-core`，本包只做旧路径重导出
  （多为 3–19 行 shim）。
- **实现已迁出**：模型厂商、文档加载器、向量库、搜索工具集等绝大多数实现已迁移到
  `langchain-community` 或独立 partner 包（`langchain-openai` 等）。本包保留
  `create_importer` 延迟导入桩 + `LangChainDeprecationWarning`。
- **活跃维护的 v1 包**在 `libs/langchain_v1/`；本域为 `langchain-classic`，无新特性。

### 跨叶共用范式

- **注册表 + 工厂**：`load_chain`（chains）、`load_agent`（agents）、`load_llm`、
  `load()`（序列化基础设施）都按 `_type`/枚举字符串查静态注册表后递归组装对象图。
- **能力缝 + 实现方**：Chain、Agent、Memory、Loader、Retriever、OutputParser、Callback
  均为抽象基类定义契约，大量具体类为实现方。
- **可选依赖懒加载**：`create_importer` 让顶层 `import langchain_classic` 不强制加载
  重第三方依赖，访问具体符号时才 import 并告警。
- **目录即实例库**：loaders（147）、llms（83）、vectorstores（75）、retrievers（44）、
  utilities（58）等目录中绝大多数文件是同构转发桩或厂商实例，分析时深读代表 + 共性列表。

### 遗留状态说明

- Chain 的 `__call__`/`run`/`acall`/`apply`、`load_chain`（0.2.13）、AgentType 枚举
  （removal=2.0.0）均标记弃用；官方引导迁移到 LCEL（`Runnable`）与 `langgraph`。
- 本域文档分析以"理解历史架构"为目的，不建议在新代码中使用。

## 图清单

本域共 10 张 archify 交互式 HTML 图（每叶至少 1 张），质量档位详见各叶第 10 节。
架构图多为 `standard`（跨层连接复杂），时序/数据流/状态机按场景补充。
