# 代理中间件体系（agent-middleware）

> 本文是 `agents` 域下的叶子子系统文档。域级总览见 `../agents.md`，本文只展开中间件的基类契约、
> 钩子模型与内置中间件清单；中间件如何被工厂装配进图、复合成洋葱链见 `../agent-factory/agent-factory.md`。
>
> 源码基准：`langchain_v1` master，commit `89252a8f7043a74f2300729fd038df8221a87e1e`，源码目录 `libs/langchain_v1/langchain/agents/middleware/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `AgentMiddleware` 基类 | 所有中间件的抽象基类，定义 6 个可覆写钩子（同步+异步） | `middleware/types.py:385` |
| `ModelRequest / ModelResponse` | 单次模型调用的输入/输出封装，中间件可 `override()` 改写 | `middleware/types.py:88,273` |
| `ExtendedModelResponse` | 携带跳转 `Command` 的扩展响应 | `middleware/types.py:291` |
| 函数式装饰器 | `@before_model / @after_model / @before_agent / @after_agent / @wrap_model_call / @wrap_tool_call / @dynamic_prompt`，把普通函数包装成中间件 | `middleware/types.py:934,1123,1300,1512,1690,1846,2019` |
| `hook_config` / `omit_payload` / `TracePolicy` | 钩子行为配置与 trace 裁剪 | `middleware/types.py:879`；`_trace_policy.py` |
| 模型重试 | `ModelRetryMiddleware`，指数退避 + jitter + 可配置异常白名单 | `middleware/model_retry.py:33` |
| 模型回退 | `ModelFallbackMiddleware`，主模型失败切备用模型 | `middleware/model_fallback.py:279` |
| 模型调用上限 | `ModelCallLimitMiddleware`，限制单轮模型调用次数 | `middleware/model_call_limit.py:94` |
| 工具重试 | `ToolRetryMiddleware`，工具调用失败重试 | `middleware/tool_retry.py:32` |
| 工具错误处理 | `ToolErrorMiddleware`，把工具异常包装为 ToolMessage | `middleware/tool_error.py:37` |
| 工具调用上限 | `ToolCallLimitMiddleware`，限制工具调用次数防死循环 | `middleware/tool_call_limit.py:141` |
| LLM 工具模拟器 | `LLMToolEmulator`，用 LLM 模拟工具行为 | `middleware/tool_emulator.py:29` |
| 工具选择 | `LLMToolSelectorMiddleware`，LLM 动态从大工具集选工具 | `middleware/tool_selection.py:123` |
| 上下文摘要 | `SummarizationMiddleware`，消息超长时摘要压缩历史 | `middleware/summarization.py:232` |
| 上下文编辑 | `ContextEditingMiddleware`，按规则编辑消息历史 | `middleware/context_editing.py:187` |
| 待办清单 | `TodoListMiddleware`，规划/跟踪任务清单 | `middleware/todo.py:174` |
| 人在回路 | `HumanInTheLoopMiddleware`，中断等待人工审批 | `middleware/human_in_the_loop.py:219` |
| Shell 工具 | `ShellToolMiddleware`，带执行策略（Host/Docker/Codex）的命令执行 | `middleware/shell_tool.py:518` |
| 文件搜索 | `FilesystemFileSearchMiddleware`，本地文件检索 | `middleware/file_search.py:108` |
| PII 检测 | `PIIMiddleware`，个人隐私信息检测与脱敏 | `middleware/pii.py:492` |
| Provider 工具搜索 | `ProviderToolSearchMiddleware`，按需从 provider 发现工具 | `middleware/provider_tool_search.py:57` |
| 共享重试工具 | `_retry.py` 的 `calculate_delay / default_retry_on / should_retry_exception / validate_retry_params` | `middleware/_retry.py` |
| 对外导出 | `middleware/__init__.py` 导出全部 16 个具体中间件 + 基类 + 装饰器 | `middleware/__init__.py:53` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `AgentMiddleware[StateT, ContextT, ResponseT]` | `types.py:385` | 基类；类属性 `state_schema / tools / trace_policy / transformers` |
| `wrap_model_call(request, handler)` | `types.py:503` | 包裹模型调用的洋葱钩子，可多次调 handler（重试）或短路 |
| `wrap_tool_call(request, handler)` | `types.py:674` | 包裹工具调用的洋葱钩子 |
| `before_model / after_model` | `types.py:455,479` | 每轮模型调用前/后的节点钩子，返回状态更新 |
| `before_agent / after_agent` | `types.py:431,650` | 整次代理执行开始前/结束后的节点钩子 |
| `hook_config(...)` | `types.py:879` | 配置钩子的 `can_jump_to` 等行为 |
| `dynamic_prompt` | `types.py:1690` | 动态系统提示词装饰器 |
| `ModelRetryMiddleware.__init__` | `model_retry.py:116` | `max_retries=2, backoff_factor=2.0, initial_delay=1.0, max_delay=60.0, jitter=True` |

## 3. 关键调用链

**调用链一：`wrap_model_call` 洋葱复合（以 ModelRetryMiddleware 为例）**

1. 工厂把各中间件的 `wrap_model_call` 按注册顺序复合成单处理器（见 agent-factory 叶子）。
2. 运行期最外层中间件先收到 `ModelRequest`，执行前置逻辑后调用 `handler(request)` 进入下一层（`model_retry.py:243`）。
3. `ModelRetryMiddleware.wrap_model_call` 进入 `for attempt in range(max_retries+1)` 循环，调 `handler(request)`（`model_retry.py:241`）。
4. 成功 → 直接返回 `ModelResponse`；捕获异常 → 跳过 `GraphBubbleUp`（图控制流异常，必须原样上抛），否则用 `should_retry_exception` 判断是否可重试（`model_retry.py:244,250`）。
5. 可重试且还有额度 → `calculate_delay` 算退避，`time.sleep` 后继续；额度用尽 → `_handle_failure`（`on_failure="continue"` 返回错误 AIMessage，`"error"` 重抛，自定义 callable 格式化）（`model_retry.py:257,269`）。

**调用链二：节点钩子（before_model/after_model）**

1. 工厂为每个实现了钩子的中间件建独立图节点（如 `MyMiddleware.before_model`）。
2. 节点执行中间件钩子函数，返回状态更新 dict。
3. `_add_middleware_edge` 按 `can_jump_to` 决定条件路由（可跳转 model/tools/end）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|------|------|
| `ModelRetryMiddleware.max_retries` | 2 | `model_retry.py:119` |
| `retry_on` | `default_retry_on`（可重试模型错误+未分类异常） | `model_retry.py:120` |
| `on_failure` | `"continue"`（返回错误 AIMessage），可选 `"error"` 或 callable | `model_retry.py:121` |
| `backoff_factor / initial_delay / max_delay / jitter` | 2.0 / 1.0s / 60s / True(±25%) | `model_retry.py:122-125` |
| 中间件 `tools` 类属性 | 中间件可注册额外工具（如 ShellToolMiddleware） | `types.py:400` |
| 中间件 `trace_policy` | None，钩子 span 正常 trace | `types.py:403` |

## 5. 错误与重试语义

- **重试白名单**：`retry_on` 可为异常元组或 callable；不在白名单的异常立即重抛，不进 `on_failure`（`model_retry.py:250`）。
- **退避**：`initial_delay * backoff_factor ** n`，封顶 `max_delay`，`jitter=True` 加 ±25% 随机抖动防惊群（`_retry.calculate_delay`）。
- **图控制流异常保护**：`GraphBubbleUp`（LangGraph 的中断/取消信号）一律原样重抛，绝不吞（`model_retry.py:244,298`）。
- **失败兜底**：`on_failure="continue"` 把错误包成 AIMessage 让代理继续；`"error"` 停止代理；callable 自定义文案。
- **参数校验**：构造期 `validate_retry_params` 校验 `max_retries>=0`、延迟非负，非法抛 `ValueError`。

## 6. 并发细节

- **同步/异步双轨**：每个钩子都有 sync 与 async 成对实现（如 `wrap_model_call` / `awrap_model_call`），工厂分别复合，按调用路径选择。
- **退避阻塞**：同步路径用 `time.sleep(delay)`，异步用 `await asyncio.sleep(delay)`（`model_retry.py:265,319`）。
- **无显式共享状态**：中间件实例在创建期配置，运行期无锁；跨轮状态经 LangGraph 状态通道（`state_schema`）传递。
- **trace 隔离**：每个 `wrap_*` 处理器经 `@traceable` 包裹，`_scrub_inputs` 剥离不可序列化对象。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `AgentMiddleware` 基类契约、6 钩子、函数式装饰器、16 个内置中间件、共享重试工具。

**Out-of-Scope（不在本仓库源码内）**
- LangGraph 状态图节点调度、`Runtime`、`Command` 执行（langgraph 包，不在本仓库源码内）。
- Shell 执行的 Docker / Codex 沙箱（外部运行时，不在本仓库源码内）。
- PII 检测的底层模型（若用外部 NLP 服务，不在本仓库源码内）。
- 不做：图装配本身（见 agent-factory）、工具执行引擎（见 agent-execution-tools）。

## 8. 与相邻子系统交互

- 上游 agent-factory → 本叶子：工厂按 6 钩子筛选中间件、复合 `wrap_*` 链、把节点钩子建成图节点。
- 本叶子 → 下游 model：`wrap_model_call` 链最内层调 `handler` 真正请求模型。
- 本叶子 → 下游 tools：`wrap_tool_call` 链注入 `ToolNode` 包裹每次工具执行。
- 本叶子 → 运行时：经 `runtime` 参数访问上下文，经 `Command`/`jump_to` 影响路由。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **能力缝归组**：中间件是"可插拔钩子链"的典型能力缝——`AgentMiddleware` 基类定义契约，16+ 实现方覆写个别钩子，工厂用"方法是否仍为基类默认"做鸭子式注册。
- **洋葱中间件**：`wrap_model_call` / `wrap_tool_call` 采用 onion 模式（外→内请求、内→外响应），`_chain_model_call_handlers` 复合。
- **双轨实现**：每个钩子 sync/async 双实现；重试中间件同步 `time.sleep`、异步 `asyncio.sleep`。
- **配置驱动**：重试/回退/上限等行为全部由中间件构造参数决定，同一钩子不同配置不同行为。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| 中间件分类架构图 | `agent-middleware-architecture.html` | architecture | showcase |
| wrap_model_call 洋葱链数据流 | `agent-middleware-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/`。本轮两图均由 standard 提升至 showcase：架构图精简为"基类→四类实现→模型"主路径；数据流图为 exec↔llm 往返流的"请求/响应"标签加 `labelDy` 垂直分离，消除 `composition/label-route-clearance` 标签重叠。
