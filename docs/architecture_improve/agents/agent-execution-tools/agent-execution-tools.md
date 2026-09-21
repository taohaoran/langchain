# 工具执行与子代理转换（agent-execution-tools）

> 本文是 `agents` 域下的叶子子系统文档。域级总览见 `../agents.md`，本文只展开 v1 的工具面再导出垫片与
> 子代理流转换器；工具节点如何被工厂装配、中间件如何包裹工具调用见 `../agent-factory/agent-factory.md` 与
> `../agent-middleware/agent-middleware.md`。
>
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，源码目录 `libs/langchain_v1/langchain/tools/` 与
> `libs/langchain_v1/langchain/agents/_subagent_transformer.py`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 工具符号再导出 | 从 `langchain_core.tools` 再导出 `BaseTool / tool / ToolException / InjectedToolArg / InjectedToolCallId` | `tools/__init__.py:3` |
| 工具运行时注入 | 从 `langgraph.prebuilt` 再导出 `InjectedState / InjectedStore / ToolRuntime`，供工具函数声明注入状态/存储 | `tools/tool_node.py:3` |
| 工具调用契约 | 再导出 `ToolCallRequest / ToolCallWithContext / ToolCallWrapper`（中间件 `wrap_tool_call` 签名所用） | `tools/tool_node.py:4` |
| `ToolNode` 占位 | 再导出 `ToolNode` 为 `_ToolNode`（供向后兼容导入） | `tools/tool_node.py:9` |
| 子代理流提升 | `SubagentTransformer` 把带 `lc_agent_name` 的嵌套命名子图提升为 `run.subagents` 上的类型化句柄 | `agents/_subagent_transformer.py:120` |
| 子代理流事件 | `SubagentRunStream / AsyncSubagentRunStream` 同步/异步子代理流视图 | `agents/_subagent_transformer.py:44,84` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `InjectedState / InjectedStore / ToolRuntime` | 来自 langgraph（本仓垫片再导出） | 工具参数注入标记：声明工具函数可访问图状态/跨线程存储 |
| `ToolCallWrapper` | 来自 langgraph.prebuilt | `wrap_tool_call` 洋葱链中单个包裹器的签名 |
| `SubagentTransformer(_TasksLifecycleBase)` | `_subagent_transformer.py:120` | 流变换器，识别子代理边界并转发子作用域事件 |
| `SubagentRunStream` | `_subagent_transformer.py:44` | 同步子代理运行流句柄 |

## 3. 关键调用链

**调用链一：工具执行（由工厂驱动）**

1. 工厂用用户工具 + 中间件工具构造 langgraph 的 `ToolNode`（见 agent-factory 叶子），注入 `wrap_tool_call` 洋葱链。
2. 模型产出 `tool_calls` 后，工厂用 `Send("tools", [call])` 为每个工具调用派发子任务。
3. `ToolNode`（在 langgraph 内，不在本仓库源码内）实际执行工具函数；`InjectedState/InjectedStore` 标记的参数由运行时注入真实状态与存储。
4. 工具结果作为 `ToolMessage` 回到消息列表。

**调用链二：子代理流提升**

1. `create_agent(name="X")` 给编译图打上 `lc_agent_name` 元数据（见 agent-factory 叶子）。
2. 该图作为子图被另一个代理调用时，`SubagentTransformer._on_started` 在任务启动时检查该命名空间是否带 `lc_agent_name`（`_subagent_transformer.py:168`）。
3. 命中则在首次启动时建子 mux，在 `run.subagents` 上发射类型化句柄，后续子作用域事件转发到该句柄（`_subagent_transformer.py:176`）。
4. 未命名（`None`）的嵌套运行被排除；同名自递归子代理仍会被提升。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|------|------|
| `SubagentTransformer(scope=())` | 作用域元组，默认根作用域 | `_subagent_transformer.py:142` |
| 子代理识别条件 | 嵌套运行带 `lc_agent_name`（由 `create_agent(name=...)` 设置） | `_subagent_transformer.py:169` |

## 5. 错误与重试语义

- 工具执行的重试/错误包装由 `ToolRetryMiddleware` / `ToolErrorMiddleware` 负责（见 agent-middleware 叶子），本叶子不直接处理。
- 子代理流变换器只做事件转发，不吞异常；子图自身错误沿 langgraph 错误通道传播。

## 6. 并发细节

- **同步/异步双流**：`SubagentRunStream` 与 `AsyncSubagentRunStream` 成对；`SubagentTransformer.supports_sync=True`，同步路径也工作。
- **事件驱动**：基于 langgraph `StreamMux` / `StreamChannel` 推送子代理流事件，无显式锁。
- **句柄去重**：`_handles` 按命名空间去重，避免重复提升。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 工具面符号再导出垫片、子代理流提升变换器。

**Out-of-Scope（不在本仓库源码内）**
- `ToolNode` 工具执行引擎、`StateGraph` 子图调度、`StreamMux`（langgraph 包，不在本仓库源码内）。
- `BaseTool / tool` 装饰器本体（langchain-core，不在本叶子源码内）。
- 不做：工具中间件逻辑（见 agent-middleware）。

## 8. 与相邻子系统交互

- 上游 agent-factory → 本叶子：工厂把 `SubagentTransformer` 与 `ToolCallTransformer` 一起注册进编译图的 transformers 列表。
- 本叶子 → 下游 langgraph：再导出符号供用户/中间件类型标注，子代理变换器挂到流 mux。
- 本叶子 → 下游工具：`InjectedState/InjectedStore` 标记让工具函数能访问图状态与存储。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **再导出垫片**：`tools/` 不重新实现工具体系，而是把 langchain-core 与 langgraph 的符号在 v1 包面统一聚合，属"前端符号面工程"。
- **流变换器扩展点**：`SubagentTransformer` 是 langgraph 流事件管道的插件，展示了如何在不侵入图运行时的情况下增强可观测性（把子代理运行提升为可消费的流）。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| 工具执行与子代理提升架构图 | `agent-execution-tools-architecture.html` | architecture | standard |
| 子代理流提升数据流 | `agent-execution-tools-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/`。本轮新增 dataflow 图：子代理流提升是一条"运行事件 → `_on_started` 拦截 → 识别 `lc_agent_name` → 首次建子 mux → 发类型化句柄"的数据管道，与静态架构图互补表达运行期事件流转。时序图不适用故省略（单次交互链路已由 agent-factory 叶子的执行循环覆盖）。
