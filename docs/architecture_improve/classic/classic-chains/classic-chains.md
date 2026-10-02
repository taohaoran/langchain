# 经典链（classic-chains）

> 本文是 `classic` 域下的叶子子系统文档（第二轮改进版）。域级总览见 `../classic.md`，本文只展开
> 经典包的 **Chain 抽象与各类内置链实现**，不展开 Agent 执行循环（见 `../classic-agents/`）、
> 记忆对象（见 `../classic-memory/`）、模型基类（见 `../classic-llms-chat/`）。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `89252a8f`。
>
> 本轮改进：保留基线两张图并按当前 archify 版本重渲染；架构图主路径与卡片已收紧，时序图补全
> 激活条。相对基线的整体改进见 `../../improve-comparison.md`。

## 1. 功能清单

本叶子实现经典包中"结构化组件调用序列"——Chain 抽象及其数十种内置实现，源码位于
`libs/langchain/langchain_classic/chains/`（140 个 py 文件）。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| Chain 抽象基类 | 所有链的基类，定义 `invoke`/`run`/`_call` 协议、输入输出校验、记忆接入、回调配置 | `chains/base.py:52`（`Chain`） |
| LLMChain | 模板提示词 + 模型 + 输出解析器的最小编排单元，`generate` 批量调用模型 | `chains/llm.py:45`（`LLMChain`） |
| 顺序链 | `SequentialChain` 多步串联、`SimpleSequentialChain` 单输出直传下一步 | `chains/sequential.py:16`、`:123` |
| 文档合并基类 | `BaseCombineDocumentsChain`：把多份文档合并进一次（或多次）LLM 调用 | `chains/combine_documents/base.py:35` |
| Stuff 合并 | 把全部文档塞进一次提示词 | `chains/combine_documents/stuff.py` |
| Map-Reduce 合并 | 逐文档映射后归约（含折叠） | `chains/combine_documents/map_reduce.py`、`reduce.py` |
| Refine 合并 | 迭代精炼：首份文档初答，后续文档逐条修正 | `chains/combine_documents/refine.py` |
| Map-Rerank 合并 | 逐文档打分取最高分答案 | `chains/combine_documents/map_rerank.py` |
| 摘要链 | `SummarizeChain` 经 LoadingCallable 选择合并策略（stuff/map_reduce/refine） | `chains/summarize/chain.py` |
| 检索问答基类 | `BaseRetrievalQA`：检索器取文档 → 交给文档合并链 | `chains/retrieval_qa/base.py:40` |
| RetrievalQA | 检索问答具体实现，`from_chain_type` 工厂 | `chains/retrieval_qa/base.py:218` |
| 路由链 | `RouterChain` 把输入路由到目的地链，`Route(destination, next_inputs)` | `chains/router/base.py:27` |
| LLM/Embedding 路由 | 用 LLM 或 embedding 相似度选择目标链 | `chains/router/llm_router.py`、`embedding_router.py` |
| 工具型链 | 数学（LLMMathChain）、Bash（LLMBashChain）、请求（LLMRequestsChain）、moderation 等 | `chains/llm_math/`、`llm_bash/`、`llm_requests.py`、`moderation.py` |
| 序列化加载 | `load_chain` / `load_chain_from_config`：按 `_type` 字符串从 YAML/JSON 反序列化 | `chains/loading.py:682`、`:702` |
| 特殊链 | HiDE、constitutional AI、graph QA、query constructor、OpenAI functions/tools 包装 | `chains/hyde/`、`constitutional_ai/`、`graph_qa/`、`openai_functions/`、`openai_tools/` |

对外导出面见 `chains/__init__.py`；序列化注册表 `type_to_loader_dict` 见 `chains/loading.py:653`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `Chain(RunnableSerializable, ABC)` | `chains/base.py:52` | 能力缝核心契约。抽象属性 `input_keys`、`output_keys`；抽象方法 `_call`；提供模板方法 `invoke` |
| `Chain._call(inputs, run_manager)` | `chains/base.py:317` | 子类必须实现的同步执行体；异步 `_acall` 默认经 `run_in_executor` 包装 `_call` |
| `Chain.invoke` | `chains/base.py:131` | 模板方法：prep_inputs → 配回调 → on_chain_start → _call → prep_outputs → on_chain_end |
| `Chain.prep_inputs` / `prep_outputs` | `chains/base.py:521` / `:471` | 记忆注入（`load_memory_variables`）与记忆落盘（`save_context`） |
| `LLMChain` | `chains/llm.py:45` | 组合 `prompt: BasePromptTemplate` + `llm: Runnable` + `output_parser`；`from_string` 便捷构造 |
| `BaseCombineDocumentsChain` | `chains/combine_documents/base.py:35` | 文档合并族基类，`combine_docs` 契约 |
| `BaseRetrievalQA` | `chains/retrieval_qa/base.py:40` | 持有 `retriever` + `combine_documents_chain` |
| `RouterChain(ABC)` / `Route(NamedTuple)` | `chains/router/base.py:27` / `:20` | 路由契约；`Route` 携带 `destination` 与 `next_inputs` |
| `type_to_loader_dict` | `chains/loading.py:653` | 字符串 `_type` → loader 函数的注册表（约 21 项） |
| `load_chain_from_config` / `load_chain` | `chains/loading.py:682` / `:702` | 序列化反序列化入口；`load_chain` 已 `@deprecated(0.2.13)` |

## 3. 关键调用链

**链的一次执行（模板方法，`Chain.invoke`，`chains/base.py:131`）**：

1. `ensure_config(config)` 归一化运行配置，取 `callbacks`/`tags`/`metadata`/`run_name`。
2. `prep_inputs(inputs)`：若输入非 dict 则按单 input_key 包成 dict；若有 `memory` 则
   `memory.load_memory_variables` 合并记忆变量（`chains/base.py:540`）。
3. `CallbackManager.configure(...)` 合并运行时与构造期回调，`on_chain_start` 发起运行。
4. `_validate_inputs` 检查 `input_keys` 齐全（`chains/base.py:289`）。
5. 反射判断 `_call` 是否接受 `run_manager`，调用子类 `_call` 得到 outputs。
6. `prep_outputs`：`_validate_outputs` → 有 memory 则 `save_context(inputs, outputs)` →
   按 `return_only_outputs` 决定是否合并输入（`chains/base.py:471`）。
7. `on_chain_end(outputs)`；异常路径 `on_chain_error(e)` 后 re-raise。

**LLMChain 内部（`LLMChain._call` → `generate`，`chains/llm.py:112`）**：
`generate([inputs])` → `prep_prompts` 调 `prompt.format_prompt` 得 `ChatPromptValue` 与 stop →
若 `llm` 是 `BaseLanguageModel` 走 `generate_prompt`，否则走 `llm.bind(stop).batch(...)` →
`create_outputs` 用 `output_parser.parse_result` 解析为 `{output_key: text}`。

**反序列化加载（`load_chain`，`chains/loading.py:702`）**：
读 YAML/JSON → `load_chain_from_config` 取 `_type` → 查 `type_to_loader_dict` →
递归地把子链配置（如 `combine_documents_chain`/`llm_chain_path`）再次 `load_chain_from_config`
或 `load_chain` 装载，最终组装出对象图。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `Chain.memory` | `None`；非空时每次运行前注入、结束后落盘 | `chains/base.py:75` |
| `Chain.verbose` | 取全局 `get_verbose()`（`chains/base.py:46`） | `chains/base.py:88` |
| `Chain.callbacks` / `tags` / `metadata` | 构造期附加，运行期可叠加且向下游传播 | `chains/base.py:82,92,98` |
| `Chain.callback_manager` | 已废弃，触发 `DeprecationWarning` 并迁移到 `callbacks` | `chains/base.py:245` |
| `LLMChain.output_parser` | `StrOutputParser` | `chains/llm.py:86` |
| `load_chain` 配置 | 必须含 `_type` 键；`lc://` 旧 GitHub Hub 协议已拒绝 | `chains/loading.py:684`、`:704` |

## 5. 错误与重试语义

- **输入校验**：`_validate_inputs` 缺键抛 `ValueError("Missing some input keys: ...")`
  （`chains/base.py:306`）；非 dict 单输入但多输入键时抛错。
- **输出校验**：`_validate_outputs` 缺输出键抛 `ValueError`（`chains/base.py:311`）。
- **回调失败传播**：执行中任何异常都会经 `run_manager.on_chain_error(e)` 记录后 **re-raise**，
  本层不做重试、不吞异常（`chains/base.py:177`）。
- **序列化失败**：`_type` 缺失/未知分别抛 `ValueError`；`sql_database_chain` 已硬编码
  `NotImplementedError` 并指向 `langchain-experimental`（`chains/loading.py:424`）。
- **`run` 限制**：多输出键时 `run` 抛 `ValueError`，仅支持单输出链（`chains/base.py:569`）。

## 6. 并发细节

本叶子以同步执行为主，异步为派生路径：

- `_acall`（`chains/base.py:340`）默认用 `run_in_executor(None, self._call, ...)` 把同步
  `_call` 丢到默认线程池，子类可覆写为真异步。
- `LLMChain.agenerate`（`chains/llm.py:138`）对真异步模型走 `agenerate_prompt`/`abatch`，
  否则仍走同步 `_call` 的 executor 包装。
- 无自建线程/锁；并发安全依赖底层 Runnable 与模型实现。
- `run_manager.get_child()` 为下游子调用派生独立回调句柄，形成运行树。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `chains/` 全部链抽象、内置实现、文档合并族、路由族、工具型链、序列化加载器。

**Out-of-Scope（不在本仓库源码内）**
- 大模型推理本身：实际 token 生成在各 `langchain-openai`/`anthropic` 等 partner 包与远端
  API，不在本仓库源码内。
- 向量检索/向量库实现：见 `classic-retrievers-stores/` 叶子与外部向量数据库。
- Agent 决策循环：见 `classic-agents/`。
- `langchain-experimental` 中的 SQL/其他实验链：`load_chain` 已显式拒绝。
- LangSmith Hub（`https://smith.langchain.com/hub`）为外部服务，本地 `lc://` 旧 Hub 已停用。

## 8. 与相邻子系统交互

- 上游：用户代码 / Agent / 序列化加载器（`chains/loading.py`）→ 构造并调用 `Chain`。
- 本叶子 → 下游：
  - `LLMChain` → `langchain_core` 的 `BasePromptTemplate`、`Runnable`（模型）、
    `BaseLLMOutputParser`（输出解析，见 `classic-utils-eval/` 的 `output_parsers/`）。
  - `RetrievalQA` → 检索器（`classic-retrievers-stores/`）+ 文档合并链。
  - 全链 → `BaseMemory`（`classic-memory/`）注入与落盘。
  - 全链 → 回调管理器（`classic-callbacks-infra/`）的 `on_chain_*` 钩子。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：Chain 是典型抽象基类能力缝——`Chain` 定义 `input_keys`/`output_keys`/`_call`
  契约，数十个具体链为实现方；`BaseCombineDocumentsChain` 与 `RouterChain` 是二级能力缝。
- **模板方法模式**：`invoke` 固定了"记忆注入→回调→_call→记忆落盘"骨架，子类只填 `_call`，
  是框架级控制反转。
- **注册表与工厂**：`type_to_loader_dict` 是字符串 `_type` → loader 的静态注册表，
  `load_chain_from_config` 为工厂；loader 内部递归组装子链，构成对象图反序列化。
- **可选依赖与懒加载**：工具型链（math/bash/requests）在 `_call` 内部延迟 import 第三方
  依赖（如 PythonREPL、requests），顶层 import 不强制加载。
- **配置驱动**：链的行为由构造配置（prompt/llm/retriever/合并策略）决定；`from_chain_type`、
  `from_llm` 工厂按 `chain_type` 字符串选择合并策略。
- **与 core/v1 的关系**：`Chain` 继承自 `langchain_core.runnables.RunnableSerializable`，
  `__call__`/`run`/`acall`/`apply` 均标记 `@deprecated(0.1.0, removal=2.0.0)`，引导迁移到
  `Runnable` LCEL 接口；`load_chain` 标记 `@deprecated(0.2.13)`。本包为遗留兼容层。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| Chain 体系与调用依赖架构图 | `classic-chains-architecture.html` | architecture | standard |
| `Chain.invoke` 一次执行时序图 | `classic-chains-sequence.html` | sequence | showcase |

档位披露：架构图因跨层连接（基类→文档合并族→路由族→模型/记忆/回调）节点较多，showcase
布局校验未全过，降为 `standard`。时序图本轮通过移除显式 viewBox 让渲染器自动适配画布宽度，
解决桌面可读性字号与视口比例检查，由 `standard` 提升至 `showcase`。
JSON IR 源文件位于 `json/`。
