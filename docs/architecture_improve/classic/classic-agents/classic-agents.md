# 经典代理执行框架（classic-agents）

> 本文是 `classic` 域下的叶子子系统文档（第二轮改进版）。域级总览见 `../classic.md`，本文只展开
> 经典包的 **Agent 决策与执行循环**，不展开 Chain 基类（见 `../classic-chains/`）、工具定义
> （见 `../classic-utils-eval/` 与 `langchain_core.tools`）、Hub 拉取（见
> `../classic-callbacks-infra/`）。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `89252a8f`。
>
> 本轮改进：架构图在当前 archify 版本下由基线 `standard` 提升至 `showcase`；状态机图保留。

## 1. 功能清单

经典 Agent 框架负责"LLM 决定下一步调哪个工具、循环执行直到给出最终答案"，源码位于
`libs/langchain/langchain_classic/agents/`（146 个 py 文件）。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 单动作 Agent 基类 | `BaseSingleActionAgent`：`plan`/`aplan` 契约，一次一步 | `agents/agent.py:55` |
| 多动作 Agent 基类 | `BaseMultiActionAgent`：一轮可返回多个动作 | `agents/agent.py:221` |
| Runnable 适配 | `RunnableAgent`/`RunnableMultiActionAgent`：把 LCEL Runnable 包成 Agent | `agents/agent.py:389`、`:497` |
| 旧式 Agent | `Agent` 基类（scratchpad 格式化 + 单动作） | `agents/agent.py:704` |
| LLMSingleActionAgent | 把 LLMChain + OutputParser 包成单动作 Agent | `agents/agent.py:615` |
| AgentExecutor | **执行循环主体**，继承 `Chain`，驱动 plan→tool→观察 循环 | `agents/agent.py:1012` |
| 错误工具 | `ExceptionTool`：工具异常包成观察回喂 LLM | `agents/agent.py:983` |
| AgentType 枚举 | 全部内置 Agent 类型字符串 | `agents/agent_types.py:15` |
| 注册表 | `AGENT_TO_CLASS`：AgentType → Agent 类 | `agents/types.py:17` |
| 工厂 initialize_agent | 按 agent_type 字符串装配 Agent+Tools+Executor | `agents/initialize.py:24` |
| 序列化加载 | `load_agent`/`load_agent_from_config` 按 `_type` 反序列化 | `agents/loading.py:39` |
| ReAct Agent | `create_react_agent`（prompt+model+tools）、旧式 ReActChain | `agents/react/agent.py:16`、`react/base.py` |
| MRKL / ZeroShot | `MRKLChain`、`ZeroShotAgent` | `agents/mrkl/base.py` |
| 对话型 Agent | `ConversationalAgent`、`ConversationalChatAgent` | `agents/conversational/`、`conversational_chat/` |
| 工具调用类 | OpenAI functions/tools、structured chat、json chat、XML、tool_calling | `agents/openai_functions_agent/`、`openai_tools/`、`structured_chat/`、`json_chat/`、`xml/`、`tool_calling_agent/` |
| 迭代器 | `AgentExecutorIterator` 可逐步遍历执行过程 | `agents/agent_iterator.py` |
| 工具加载 | `load_tools` 按名字字符串加载工具集 | `agents/load_tools.py`、`tools.py` |
| 工具包 | `agent_toolkits/`（SQL、OpenAPI、矢量库等工具包） | `agents/agent_toolkits/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `BaseSingleActionAgent(BaseModel)` | `agents/agent.py:55` | 能力缝：`plan(intermediate_steps, ...)` 返回 `AgentAction \| AgentFinish` |
| `BaseMultiActionAgent` | `agents/agent.py:221` | 一轮多动作变体，`plan` 返回动作列表 |
| `AgentExecutor(Chain)` | `agents/agent.py:1012` | 执行循环，持有 `agent`、`tools`、`memory`、各类停机/容错配置 |
| `AgentAction` / `AgentFinish` | 来自 `langchain_core` schema | 决策结果：调工具（带 tool/tool_input）或结束（带 return values） |
| `RunnableAgent` | `agents/agent.py:389` | 把 `Runnable` 的 invoke 结果适配为 plan 输出 |
| `AgentType` (str Enum) | `agents/agent_types.py:15` | 9+ 种内置 Agent 类型标识 |
| `AGENT_TO_CLASS` | `agents/types.py:17` | AgentType → 类的静态注册表 |
| `initialize_agent` | `agents/initialize.py:24` | 高层工厂：校验 agent_type、查注册表、构造 executor |
| `handle_parsing_errors` | `agents/agent.py:1042` | 输出解析失败时的容错策略（bool/str/Callable） |

## 3. 关键调用链

**AgentExecutor 执行循环（`AgentExecutor._call`，`agents/agent.py:1570`）**：

1. 建 `name_to_tool_map`（工具名→工具）与颜色映射；`intermediate_steps=[]` 空。
2. 进入 `while self._should_continue(iterations, time_elapsed)` 循环：
   - `_take_next_step(...)`：调用 `agent.plan(intermediate_steps)` 得到下一步输出。
   - 若结果是 `AgentFinish` → `_return(...)` 收尾返回。
   - 否则把 `(action, observation)` 追加进 `intermediate_steps`。
   - `_get_tool_return` 判断是否有工具配置了直接返回，是则提前结束。
   - `iterations += 1`，更新耗时。
3. 循环因 `max_iterations`/`max_execution_time` 退出时，调
   `agent.return_stopped_response(early_stopping_method, ...)` 产出停机答案。

**异步路径（`_acall`，`agents/agent.py:1619`）**：用 `asyncio_timeout(self.max_execution_time)`
包住循环，`_atake_next_step`/`_areturn` 为异步版，逻辑与同步一致。

**装配路径（`initialize_agent`，`agents/initialize.py:24`）**：
校验 `agent_type ∈ AGENT_TO_CLASS` → 取 `agent_cls` → 构造 agent 与 tools → 包进
`AgentExecutor`。`load_agent` 走 `load_agent_from_config` 取 `_type` 查同一注册表
（`agents/loading.py:76`）。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `max_iterations` | `15`；达到后停机 | `agents/agent.py:1023` |
| `max_execution_time` | `None`；异步循环用 `asyncio_timeout` 包裹 | `agents/agent.py:1028` |
| `early_stopping_method` | `"force"`；可选 `"generate"`（再让 LLM 生成收尾） | `agents/agent.py:1032` |
| `handle_parsing_errors` | `False`；可传 str 或 `Callable[[OutputParserException], str]` | `agents/agent.py:1042` |
| `return_intermediate_steps` | `False`；是否在输出中附带中间步骤 | `agents/agent.py:1020` |
| `memory` | 继承自 `Chain`，对话型 Agent 常挂记忆 | `agents/agent.py` |
| AgentType 注册表 | 9 种内置类型，新增 Agent 须登记 `AGENT_TO_CLASS` | `agents/types.py:17` |

## 5. 错误与重试语义

- **工具异常**：`ExceptionTool`（`agents/agent.py:983`）把工具执行异常包成一条 observation
  回喂给 LLM，让 Agent 自行纠错，而非直接崩溃。
- **输出解析错误**：`handle_parsing_errors` 决定策略——`False` 直接抛出；`True`/str 时把错误
  作为观察回喂；Callable 允许自定义纠错消息。
- **停机**：超过 `max_iterations`/`max_execution_time` 不抛错，而是走
  `return_stopped_response`（force 强制返回最近结果 / generate 再让模型总结）。
- **未知 Agent 类型**：`initialize_agent` 与 `load_agent_from_config` 抛 `ValueError` 并列出
  合法类型（`agents/initialize.py:80`、`loading.py:78`）。
- 本层不做工具调用重试；重试/退避由具体工具或模型层负责。

## 6. 并发细节

- **执行循环**：同步 `_call` 在单线程内串行 plan→tool→observe；无 fan-out（多动作在同一
  步内顺序执行）。
- **异步**：`_acall` 用 `asyncio_timeout` 做整体超时；多动作 `_atake_next_step` 内部可并发
  执行独立工具调用（依 `BaseMultiActionAgent` 实现）。
- **迭代器**：`AgentExecutorIterator`（`agents/agent_iterator.py`）把循环暴露为可逐步拉取的
  迭代器，便于流式观察，不改变并发模型。
- 无显式锁；`intermediate_steps` 在单循环内单线程读写。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `agents/` 的 Agent 基类、AgentExecutor 循环、各类内置 Agent 实现、工厂与序列化加载、
  工具加载与工具包。

**Out-of-Scope（不在本仓库源码内）**
- 工具的实际实现（搜索、计算、API 调用）多在 `langchain-community` 或外部服务。
- 工具/模型的网络调用、远端 LLM API，不在本仓库源码内。
- LangSmith Hub（`hub.pull("hwchase17/react")`）为外部服务（`react/agent.py:68`）。
- 新一代 `langgraph` 图式 Agent 执行器不在本仓库；本框架为遗留控制流。

## 8. 与相邻子系统交互

- 上游：用户代码 / `initialize_agent` / `load_agent` → 装配并调用 `AgentExecutor`。
- 本叶子 → 下游：
  - `AgentExecutor` → `Chain` 基类（`classic-chains/`）复用回调与记忆骨架。
  - `plan` → 模型 Runnable（`classic-llms-chat/` 与 partner 包）。
  - 工具执行 → `langchain_core.tools.BaseTool` 与各工具包。
  - 记忆读写 → `classic-memory/`。
  - 提示词从 Hub 拉取 → `hub.py`（`classic-callbacks-infra/`）。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseSingleActionAgent`/`BaseMultiActionAgent` 是决策契约能力缝；
  `AgentExecutor` 是"控制循环"能力缝（plan 策略可插拔）。`RunnableAgent` 是适配器，把新
  LCEL 风格 Runnable 桥接到旧 Agent 契约。
- **注册表与工厂**：`AGENT_TO_CLASS` 是 AgentType 枚举 → 类的静态注册表；`initialize_agent`
  与 `load_agent_from_config` 是工厂，按字符串/枚举路由。
- **隐藏状态机在循环里**：AgentExecutor 的 while 循环（plan → tool → observe → 再 plan）是
  明确状态机，配 lifecycle 图表达。
- **可选依赖与懒加载**：`load_tools` 与 `agent_toolkits` 按字符串延迟 import 第三方工具。
- **与 core/v1 的关系**：Agent 类型枚举整体标注 `AGENT_DEPRECATION_WARNING`、`removal="2.0.0"`；
  官方引导迁移到 `langgraph`。`AgentExecutor` 继承自遗留 `Chain`。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| Agent 体系与执行架构图 | `classic-agents-architecture.html` | architecture | showcase |
| ReAct 执行循环状态机 | `classic-agents-lifecycle.html` | lifecycle | standard |

档位披露：架构图在当前 archify 版本下通过 showcase 校验（相对基线 `standard` 为提升项）。
状态机因循环回边与终态带列约束（outcome band 仅允许列 0..2）需显式 via 绕线，showcase 的
微段间距门槛未过，降为 `standard`。JSON IR 位于 `json/`。
