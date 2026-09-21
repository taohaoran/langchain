# agents 域总览

> 本域包含以下 6 个叶子子系统；各叶子详情见对应文档。
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，源码目录 `libs/langchain_v1/langchain/`（约 40 个 py 文件 / 1.6 万行）。

## 1. 域职责

agents 域是 `langchain_v1`（当前活跃维护的 langchain 包）的核心，围绕**代理编排**：以 `create_agent` 工厂
把聊天模型、工具、中间件组装成一张 LangGraph 状态图，并提供可插拔中间件体系、MCP 协议适配、模型初始化与
消息/限流符号面。设计范式是"工厂 + 中间件钩子链 + 配置驱动"。

## 2. 叶子索引

| 叶子 | 文档 | 架构图 | 时序/数据流/其他 | 职责一句话 |
|------|------|--------|-------------|-----------|
| agent-factory | [agent-factory.md](agent-factory/agent-factory.md) | [架构图](agent-factory/agent-factory-architecture.html) | [循环状态机](agent-factory/agent-factory-lifecycle.html) | `create_agent` 工厂组装模型/工具/中间件为图 |
| agent-middleware | [agent-middleware.md](agent-middleware/agent-middleware.md) | [架构图](agent-middleware/agent-middleware-architecture.html) | [洋葱链数据流](agent-middleware/agent-middleware-dataflow.html) | 可插拔中间件钩子体系与 16 个内置中间件 |
| agent-execution-tools | [agent-execution-tools.md](agent-execution-tools/agent-execution-tools.md) | [架构图](agent-execution-tools/agent-execution-tools-architecture.html) | [子代理流数据流](agent-execution-tools/agent-execution-tools-dataflow.html) | 工具符号再导出与子代理流提升 |
| mcp-integration | [mcp-integration.md](mcp-integration/mcp-integration.md) | [架构图](mcp-integration/mcp-integration-architecture.html) | [调用时序](mcp-integration/mcp-integration-sequence.html) | MCP 服务器工具适配为 LangChain 工具 |
| chat-models-embeddings | [chat-models-embeddings.md](chat-models-embeddings/chat-models-embeddings.md) | [架构图](chat-models-embeddings/chat-models-embeddings-architecture.html) | [调用链时序](chat-models-embeddings/chat-models-embeddings-sequence.html) | 字符串模型初始化工厂与嵌入封装 |
| messages-rate-limiters | [messages-rate-limiters.md](messages-rate-limiters/messages-rate-limiters.md) | [架构图](messages-rate-limiters/messages-rate-limiters-architecture.html) | [裁剪限流数据流](messages-rate-limiters/messages-rate-limiters-dataflow.html) | 消息类型与速率限制器符号聚合 |

## 3. 域级机制细节

- **create_agent 整体流程**：字符串模型经 `init_chat_model` 实例化 → 处理 `response_format` 三策略 → 收集工具构造 `ToolNode` → 按 6 类钩子筛选并复合 `wrap_model_call`/`wrap_tool_call` 洋葱链 → 合并状态 schema → 建 `StateGraph` 加 model/tools/中间件节点 → 装配条件边（model_to_tools/tools_to_model）→ `compile` 返回。
- **中间件链模式**：每个中间件覆写基类 `AgentMiddleware` 的 6 个钩子（before_agent/before_model/wrap_model_call/after_model/wrap_tool_call/after_agent，均同步+异步）；工厂用"方法是否仍为基类默认"做鸭子式注册；`wrap_*` 采用洋葱复合（外→内请求、内→外响应）。
- **代理循环**：model 产出 tool_calls → `Send("tools")` 并发执行工具 → 回 model，直到无新工具调用；`recursion_limit=9999`。
- **本轮增强**：为基线仅 1 图的三个叶子（agent-execution-tools、chat-models-embeddings、messages-rate-limiters）按核心语义各补第 2 张图——子代理流提升补 dataflow、`init_chat_model` 补 sequence、请求期裁剪限流补 dataflow；新增本域两张域级图。

## 4. 域级图

本域共 3 张域级图，分别覆盖三种不重复的核心语义：静态拓扑（architecture）、构建到执行的分步流程（workflow）、运行期多方消息嵌套时序（sequence）。

![agents 域架构图](agents-architecture.html)

![构建 agent 到执行一轮流程](agents-workflow.html)

![中间件洋葱链一轮交互时序](agents-onion-sequence.html)

洋葱链时序图说明：`wrap_model_call` 把各中间件按注册顺序复合成单一洋葱处理器——请求自外层向内逐层传递（`handler(request)`），最内层真正请求聊天模型；响应自内向外逐层返回。重试/回退等中间件在外层捕获模型异常后可重新调用 `handler`（重试）或切备用模型，此分支在 agent-middleware 叶子 MD 第 3 节文字展开。
