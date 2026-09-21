# 代理工厂（agent-factory）

> 本文是 `agents` 域下的叶子子系统文档。域级总览见 `../agents.md`，本文只展开 `create_agent`
> 工厂如何把模型、工具、中间件组装成一张可执行的 LangGraph 状态图，不重复展开中间件内部实现
> （见 `../agent-middleware/agent-middleware.md`）与工具节点细节（见 `../agent-execution-tools/agent-execution-tools.md`）。
>
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，源码目录 `libs/langchain_v1/langchain/agents/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `create_agent` 工厂 | 代理创建唯一对外入口，接收模型/工具/中间件/结构化输出配置，返回编译好的 `CompiledStateGraph` | `agents/factory.py:840` |
| 模型初始化 | 字符串模型标识（如 `"anthropic:claude-...`）经 `init_chat_model` 解析为 `BaseChatModel` | `agents/factory.py:995` |
| 系统提示词归一 | `str` / `SystemMessage` 统一转为 `SystemMessage`，调用模型时前置到消息列表 | `agents/factory.py:999` |
| 结构化输出装配 | 原始 schema 自动包装为 `AutoStrategy`，创建期先降级为 `ToolStrategy` 挂输出工具，运行期按模型能力再决定是否切 `ProviderStrategy` | `agents/factory.py:1010` |
| 中间件钩子收集 | 按 `before_agent / before_model / wrap_model_call / after_model / wrap_tool_call / after_agent` 六类钩子筛选实现了钩子的中间件 | `agents/factory.py:1111` |
| `wrap_model_call` 洋葱链 | 把所有 `wrap_model_call` 处理器用 `_chain_model_call_handlers` 复合成单一同步处理器，异步走 `_chain_async_model_call_handlers` | `agents/factory.py:263,355,1154` |
| `wrap_tool_call` 洋葱链 | 复合所有工具调用包裹器，注入 `ToolNode` | `agents/factory.py:658,706,1050` |
| 状态 schema 合并 | 中间件各自的 `state_schema` 按注册顺序合并，用户 `state_schema` 放最后以覆盖字段冲突 | `agents/factory.py:1180` |
| 图节点与边装配 | 动态确定 `entry_node / loop_entry_node / loop_exit_node / exit_node`，按是否有工具、是否有中间件选择不同边拓扑 | `agents/factory.py:1647` |
| 代理循环条件边 | `model_to_tools`（是否继续工具循环）、`tools_to_model`（工具后回模型）、`model_to_model`（纯结构化输出重试） | `agents/factory.py:1923,1977,2004` |
| 编译与配置 | `graph.compile(...)` 注入 checkpointer/store/interrupt/cache，设置 `recursion_limit=9999` | `agents/factory.py:1831,1841` |
| 对外导出 | `agents/__init__.py` 仅导出 `create_agent` 与 `AgentState` | `agents/__init__.py:3` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `create_agent(...)` | `agents/factory.py:840` | 工厂主函数，13 个关键字参数，返回编译后状态图 |
| `ModelRequest[ContextT]` | `middleware/types.py:88` | 单次模型调用的输入封装（model/tools/system_message/messages/state/runtime），中间件可经 `.override()` 改写 |
| `ModelResponse[ResponseT]` | `middleware/types.py:273` | 模型调用输出（result 消息列表 + structured_response） |
| `ExtendedModelResponse[ResponseT]` | `middleware/types.py:291` | 用户侧扩展响应，可携带跳转 `Command` |
| `AgentState[ResponseT]` | `middleware/types.py:349` | 代理状态 TypedDict 基类（messages、可选 structured_response、jump_to） |
| `_ComposedExtendedModelResponse` | `agents/factory.py:68` | 内部组合结果，累积各中间件层产生的 `Command` 列表 |
| `ToolStrategy / ProviderStrategy / AutoStrategy` | `structured_output.py:196,271,457` | 三种结构化输出策略：工具调用式、provider 原生响应式、自动探测式 |
| `OutputToolBinding` | `structured_output.py:318` | 把响应 schema 转成 LangChain 输出工具（供 ToolStrategy 使用） |
| `RunnableCallable` | 来自 langgraph | 同时包装同步 `model_node` 与异步 `amodel_node`，按模型能力选择执行路径 |
| `_chain_model_call_handlers` | `agents/factory.py:263` | 把有序处理器列表复合为洋葱式单处理器（外→内调用，内→外返回） |

## 3. 关键调用链

**调用链一：`create_agent` 装配阶段（构造一次）**

1. `create_agent(model, tools, middleware, ...)` 进入（`factory.py:840`）。
2. 字符串模型经 `init_chat_model` 实例化（`factory.py:995`）。
3. 处理 `response_format`：原始 schema 包成 `AutoStrategy`，再临时转 `ToolStrategy` 以在创建期计算输出工具（`factory.py:1014`）。
4. 收集中间件工具 `middleware_tools` 与客户端工具，构造 `ToolNode`，并把 `wrap_tool_call` 洋葱链注入（`factory.py:1077`）。
5. 校验中间件名字唯一（`factory.py:1108`），按六类钩子筛出对应中间件列表。
6. 复合 `wrap_model_call` 同步/异步洋葱链（`factory.py:1154`）。
7. 合并状态 schema，构造 `StateGraph`（`factory.py:1182`）。
8. 添加 `model` 节点（`RunnableCallable(model_node, amodel_node)`）与可选 `tools` 节点（`factory.py:1543`）。
9. 为每个实现了钩子的中间件添加图节点（before_agent/before_model/after_model/after_agent 各成节点）。
10. 动态确定四个路由节点，添加条件边，`graph.compile(...)` 返回（`factory.py:1841`）。

**调用链二：代理运行循环（每次 invoke）**

1. `START → entry_node`（首个 before_agent 或 before_model，否则直接 `model`）。
2. `model_node` 构造 `ModelRequest`，若有 `wrap_model_call` 链则走洋葱链，否则直接 `_execute_model_sync` 调模型（`factory.py:1468`）。
3. 模型返回后经 `model_to_tools` 条件路由：无 tool_calls 或已有 structured_response → `exit_node`；有未完成 tool_calls → `Send("tools", ...)` 并发执行工具（`factory.py:1929`）。
4. `tools` 节点执行工具后经 `tools_to_model` 路由：全部 `return_direct` 或结构化工具已执行 → 退出；否则回到 `loop_entry_node` 继续下一轮（`factory.py:2011`）。
5. 循环直到无新工具调用，最后经 after_agent 链到 `END`。

## 4. 配置项

| 参数 | 默认 / 行为 | 位置 |
|------|------|------|
| `model` | 必填，字符串或 `BaseChatModel` 实例 | `factory.py:841` |
| `tools` | `None` → 空工具集，仅模型节点无工具循环 | `factory.py:842` |
| `system_prompt` | `None`，非系统消息前置 | `factory.py:844` |
| `middleware` | `()` 空元组 | `factory.py:845` |
| `response_format` | `None`；原始 schema 自动包 `AutoStrategy` | `factory.py:846` |
| `state_schema / context_schema` | `None`，回退 `AgentState` / 无 context | `factory.py:847` |
| `checkpointer / store` | `None`，无记忆 / 无跨线程存储 | `factory.py:849` |
| `interrupt_before / interrupt_after` | `None` | `factory.py:851` |
| `recursion_limit` | 硬编码 `9999`（规避 langgraph 默认 25 步限制） | `factory.py:1831` |
| 结构化输出回退模型名单 | `FALLBACK_MODELS_WITH_STRUCTURED_OUTPUT` 正则列表（gpt-4.1/4o/5、claude、grok 等），无 profile 数据时按模型名兜底判定原生结构化输出支持 | `factory.py:174` |

## 5. 错误与重试语义

- **动态工具未知错误**：中间件在 `wrap_model_call` 里新增了未在 `create_agent` 注册的工具，运行期抛出 `DYNAMIC_TOOL_ERROR_TEMPLATE` 并列出已注册工具与未知工具名（`factory.py:119`）。
- **结构化输出校验失败**：`StructuredOutputValidationError`，由 `_handle_structured_output_error` 把错误回填给模型令其自我修正，文案为 `"Error: {error}\n Please fix your mistakes."`（`factory.py:117,628`）。
- **中间件 `Command` 限制**：`wrap_model_call` 中产生的 `Command.goto / resume / graph` 不被支持，立即 `NotImplementedError`，提示改用 `jump_to` 状态字段（`factory.py:244`）。
- **重复中间件**：同名中间件实例重复传入 → `AssertionError`（`factory.py:1108`）。
- **模型异常**：`_execute_model_sync` 不吞异常，模型调用错误直接向上抛出，重试/回退由 `model_retry` / `model_fallback` 中间件在洋葱链外层负责（见 agent-middleware 叶子）。

## 6. 并发细节

- **同步/异步双轨**：`model_node` 与 `amodel_node` 成对实现，经 `RunnableCallable` 按调用方（sync/async）选择，`wrap_model_call` 与 `awrap_model_call` 各自独立复合洋葱链（`factory.py:1441,1491,1543`）。
- **工具并发**：`model_to_tools` 用 `Send("tools", [tool_call])` 为每个待执行工具调用派发独立子任务，工具节点并行执行（`factory.py:1964`）。
- **无显式锁**：装配期是纯函数式组装，运行期并发由 LangGraph 运行时调度，本层不持临界区。
- **trace 上下文**：每个中间件 `wrap_*` 处理器经 `@traceable` 包裹并 `_scrub_inputs` 剥离不可序列化的 `handler/runtime`，trace policy 在调用期解析（`factory.py:158`）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `create_agent` 工厂、结构化输出策略三态、洋葱链复合、图拓扑装配与条件路由。
- `structured_output.py` 的策略与 schema 绑定。

**Out-of-Scope（不在本仓库源码内）**
- LangGraph 状态图运行时、`StateGraph / ToolNode / Send / Command` 执行引擎（langgraph 包，不在本仓库源码内）。
- 具体聊天模型实现（OpenAI/Anthropic 等，在 `libs/partners/` 或外部仓，不在本叶子源码内）。
- 中间件内部逻辑（见 agent-middleware 叶子）。
- 不做：工具执行细节（见 agent-execution-tools）、MCP 协议适配（见 mcp-integration 叶子）。

## 8. 与相邻子系统交互

- 上游 → 本叶子：用户代码调用 `create_agent(...)`，传入模型标识/工具/中间件。
- 本叶子 → 下游中间件：把 `middleware` 列表按钩子分类并复合，产出注入图节点与 `wrap_*` 链。
- 本叶子 → 下游工具：构造 `ToolNode` 作为图节点，`Send` 派发工具调用。
- 本叶子 → 下游 MCP：MCP 工具在用户侧先经 `mcp/adapter.py` 转成 LangChain 工具再传入 `tools=`（见 mcp-integration 叶子）。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **能力缝归组**：本叶子是"工厂 + 配置驱动"范式——`create_agent` 本身是工厂函数，`response_format` 的 `AutoStrategy→ToolStrategy/ProviderStrategy` 切换是配置驱动的静态分派（运行期按模型能力决定策略）。
- **注册表与钩子链**：中间件通过覆写基类 `AgentMiddleware` 的 6 个钩子方法实现扩展，工厂用"方法是否仍为基类默认实现"来判定中间件是否接入某钩子（鸭子式注册表，`factory.py:1111`）。
- **隐藏状态机在循环里**：代理执行循环（model→tools→model）是一个隐藏在条件边里的状态机，本叶子补 `lifecycle` 图表达其状态迁移。
- **双轨实现**：同步 `model_node` 与异步 `amodel_node` 双轨并存，经 `RunnableCallable` 统一。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| 代理工厂装配架构图 | `agent-factory-architecture.html` | architecture | showcase |
| 代理执行循环状态机 | `agent-factory-lifecycle.html` | lifecycle | showcase |

JSON IR 源文件位于 `json/`。降档说明：两张图均为精简主路径（主节点 ≤8），目标 showcase 档。
