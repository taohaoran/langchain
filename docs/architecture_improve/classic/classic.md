# classic 域：langchain_classic 经典包（第二轮改进版）

> 本文是 `docs/architecture_improve/` 下 `classic` 域的总览（第二轮迭代分析）。系统级总览由
> 汇总者负责；本域基线为 `../architecture/classic/`（只读，未改动），本轮产物全部位于
> `docs/architecture_improve/classic/`。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `89252a8f`，
> 源码位于 `libs/langchain/langchain_classic/`（约 1300 个 py 文件）。

## 1. 域职责

`langchain-classic` 是 LangChain 历史包的**遗留兼容层**。README 明确定位为「遗留 Chain、langchain-community 再导出、索引 API、已废弃功能」（原文：`Legacy chains, langchain-community re-exports, indexing API, deprecated functionality`），
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

![classic 域总体架构图](classic-architecture.html)

![经典 Chain 数据管线](classic-dataflow.html)

![经典 Chain 内 LLM 调用链时序](classic-sequence.html)

## 2. 叶子索引表

| # | 叶子 | 职责 | 主要源码 | 图（数量） |
| --- | --- | --- | --- | --- |
| 1 | [classic-chains](classic-chains/classic-chains.md) | Chain 抽象与文档合并/路由/工具型链 | `chains/` | 2（架构+时序） |
| 2 | [classic-agents](classic-agents/classic-agents.md) | Agent 决策契约与 ReAct 执行循环 | `agents/` | 2（架构+状态机） |
| 3 | [classic-llms-chat](classic-llms-chat/classic-llms-chat.md) | LLM/ChatModel 基类与厂商转发层 | `llms/`、`chat_models/`、`base_language.py` | 2（架构+导入委托时序） |
| 4 | [classic-memory](classic-memory/classic-memory.md) | 对话记忆抽象与持久化后端 | `memory/`、`base_memory.py` | 2（架构+对话读写时序） |
| 5 | [classic-loaders](classic-loaders/classic-loaders.md) | 文档加载器集合与文本分割 | `document_loaders/`、`document_transformers/`、`text_splitter.py` | 2（架构+数据流） |
| 6 | [classic-retrievers-stores](classic-retrievers-stores/classic-retrievers-stores.md) | 检索器、向量存储、键值存储、索引 | `retrievers/`、`vectorstores/`、`docstore/`、`storage/`、`indexes/` | 2（架构+检索数据流） |
| 7 | [classic-utils-eval](classic-utils-eval/classic-utils-eval.md) | 输出解析器、评估框架、工具集转发 | `output_parsers/`、`prompts/`、`evaluation/`、`utilities/` | 2（架构+重试解析时序） |
| 8 | [classic-callbacks-infra](classic-callbacks-infra/classic-callbacks-infra.md) | 回调、缓存、Hub、序列化基础设施 | `callbacks/`、`cache.py`、`hub.py`、`load/`、`globals.py` | 2（架构+事件分发数据流） |

## 3. 域级机制细节

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

## 4. 本轮相对基线的改进

- **图数量补齐**：基线仅 10 张图（6 叶各仅 1 张）；本轮为 6 个单图叶子各补第 2 张图，
  8 叶均达 ≥2 张；域级图按上调后配额补至 3 张（架构 + 数据管线 + LLM 调用链时序）。
- **质量档位显著提升**：本轮修复 archify 新版 `meta.output` 必填字段后重渲染全部 19 图。
  通过移除显式 `viewBox`（让渲染器自动适配画布宽度以解决桌面可读性字号与视口比例检查）
  及缩短过长子标签，共 7 张图由 `standard` 提升至 `showcase`：
  chains 时序、loaders 数据流、memory 时序、retrievers 数据流、utils-eval 时序、
  域级数据管线、域级 LLM 调用链时序。
- **已知薄弱项修复**：第二轮的 chains 时序与 loaders 数据流两张落 `standard` 的图，
  本轮均提升至 `showcase`。
- **新增图语义**：导入委托时序、对话记忆读写时序、加载/分割架构、检索数据流、重试解析
  时序、回调事件分发数据流、域级 LLM 调用链时序——均按 diagram-policy 核心语义选型。
- **残留 standard 披露**：6 张图因跨层连接复杂（chains 架构、域级架构）或循环回边约束
  （agents 生命周期、callbacks 事件分发数据流）保持 `standard`，属预期内复杂度。
  详见各叶第 10 节。

本域共 19 张 archify 交互式 HTML 图（域级 3 + 叶子级 16），其中 13 张 `showcase`、
6 张 `standard`，质量档位详见各叶第 10 节。
域级三图分工：架构图表达静态组件拓扑，数据管线表达"输入→编排→生成→输出"数据流向，
时序图表达"LLMChain → BaseLLM → 厂商实现 → 远端 API → OutputParser"的单次调用链，
三者互补不重复。JSON IR 源文件位于各目录 `json/` 与本目录 `json/`。
