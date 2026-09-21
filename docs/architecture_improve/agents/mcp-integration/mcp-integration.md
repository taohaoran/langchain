# Model Context Protocol 集成（mcp-integration）

> 本文是 `agents` 域下的叶子子系统文档。域级总览见 `../agents.md`，本文只展开 v1 如何把 MCP 服务器工具
> 适配为 LangChain 工具；工具如何被 `create_agent` 装配见 `../agent-factory/agent-factory.md`。
>
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，源码目录 `libs/langchain_v1/langchain/mcp/`。
> 注意：`langchain.mcp` 为 beta 命名空间，import 时发一次 `LangChainBetaWarning`（`mcp/__init__.py:26`）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `MCPAdapter` | 把 MCP 目标（URL/本地脚本/进程内 server/已有 client）适配为 LangChain 工具集 | `mcp/adapter.py:125` |
| 传输推断 | 委托 `fastmcp.Client` 推断 stdio/SSE/HTTP 传输；字符串 target 必须是 http(s) URL（安全约束） | `mcp/adapter.py:136,196` |
| 上下文管理 | 异步上下文管理器 `async with MCPAdapter(...)` 管理连接生命周期 | `mcp/adapter.py:207` |
| `list_tools()` | 发现远端工具并逐个经 `as_langchain_tool` 转换，支持缓存模式 use/refresh/bypass | `mcp/adapter.py:221` |
| `as_langchain_tool` | 把单个 MCP 工具封装为 LangChain `StructuredTool`，异步调用对应 MCP 工具 | `mcp/tools.py:212` |
| 内容块转换 | MCP 内容块 → LangChain `ToolMessageContentBlock` | `mcp/tools.py:120` |
| 工具错误处理 | MCP 业务失败转为 `status="error"` 的 ToolMessage；传输失败/不可转换内容直接上抛 | `mcp/tools.py:104,168` |
| 中断式 elicitation | 服务器中途要输入时用 LangGraph `interrupt()` 暂停，人回答后 resume | `mcp/elicitation.py`；`adapter.py:146` |
| elicitation 类型 | `MCPElicitationInterrupt/Accept/Decline/Cancel/Resume` 等 TypedDict 判别负载 | `mcp/elicitation.py:94,108,121,131,147` |
| 能力宣告 | `_declare_elicitation_capability` 向 server 宣告支持 elicitation | `mcp/elicitation.py:228` |
| 对外导出 | `MCPAdapter / as_langchain_tool / MCPToolArtifact` | `mcp/__init__.py:19` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `MCPAdapter` | `adapter.py:125` | 适配门面，持 `FastMCPClient | ClientGroup` |
| `as_langchain_tool(tool, client)` | `tools.py:212` | 单工具转换器，返回 `StructuredTool` |
| `MCPToolArtifact` | `tools.py:59` | 工具调用附带的产物 TypedDict |
| `StructuredTool` | 来自 langchain-core | 转换产物的工具类型（`response_format="content_and_artifact"`） |
| `MCPElicitationInterrupt` | `elicitation.py:94` | 中断负载 TypedDict |
| `_arm_for_interrupts(client)` | `elicitation.py:246` | 给 client 装上中断驱动的 elicitation 处理器 |

## 3. 关键调用链

**调用链一：MCP 工具发现与适配**

1. 用户 `async with MCPAdapter("https://...")` 进入上下文，构造 `FastMCPClient` 并 `_arm_for_interrupts`（`adapter.py:198`）。
2. `await adapter.list_tools()` 调 `client.list_tools()` 发现远端工具（`adapter.py:241`）。
3. 对每个远端 `Tool` 调 `as_langchain_tool(tool, client)`（`adapter.py:242`）。
4. `as_langchain_tool` 构造内部 `call_tool` 协程，返回 `StructuredTool`（`tools.py:274`）。
5. 把返回的 LangChain 工具列表传给 `create_agent(tools=...)`。

**调用链二：一次 MCP 工具调用（含 elicitation）**

1. 代理执行封装好的 LangChain 工具 → 内部 `call_tool(**arguments)`（`tools.py:261`）。
2. `async with client` 打开连接，若 client 被 arm 为中断驱动 → `_call_tool_with_interrupts`，否则普通 `client.call_tool(..., raise_on_error=False)`（`tools.py:266`）。
3. 服务器若中途请求输入 → LangGraph `interrupt()` 暂停，人在回路回答 → resume（见 elicitation 类型）。
4. 返回结果经 `_convert_call_tool_result` 转为内容块 + artifact（`tools.py:272`）。
5. 业务失败 → 错误 ToolMessage 回模型自我修正；传输失败 → 直接上抛。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|------|------|
| `MCPAdapter(target)` | 字符串必须是 http(s) URL（拒绝本地脚本路径防误启动子进程） | `adapter.py:136,196` |
| `list_tools(cache_mode="use")` | use 读缓存/refresh 强制刷新/bypass 绕过 | `adapter.py:221` |
| elicitation | 默认 arm 每个 client 宣告 elicitation 能力；已自带处理器的 client 被尊重、克隆不改写 | `adapter.py:183` |
| 已有 client | 若自带 `_elicitation_callback` 则不覆盖 | `adapter.py:184` |

## 5. 错误与重试语义

- **业务失败**：MCP 工具运行后报告失败 → 转 `ToolMessage(status="error")` 携带服务器原错误内容，模型可自我修正重试（`tools.py:230`）。
- **传输/不可转换错误**：直接上抛异常，因为模型无法据此行动（`tools.py:233`）。
- **call_tool 用 `raise_on_error=False`**：保留 MCP 错误结果供转换，不中途抛（`tools.py:271`）。
- **target 校验**：非 http(s) 的字符串 target 经 `_validate_url_target` 拒绝（`adapter.py:84`）。

## 6. 并发细节

- **异步为主**：整个 MCP 适配是异步栈（`async with` / await），工具协程 `call_tool`。
- **可重入 client**：FastMCP client 可重入，工具即便在别处已持连接也能自行打开（`tools.py:219`）。
- **不突变调用方**：arm elicitation 前先 `client.new()` 克隆，调用方对象不被改写（`adapter.py:186`）。
- **中断恢复**：elicitation 经 LangGraph `interrupt()`/resume 实现，无自定义线程同步。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `MCPAdapter`、`as_langchain_tool`、内容块转换、elicitation 中断类型与 arm 逻辑。

**Out-of-Scope（不在本仓库源码内）**
- FastMCP 协议客户端与传输（stdio/SSE/HTTP）——`fastmcp` 包，不在本仓库源码内。
- 远端 MCP 服务器本身（外部服务）。
- LangGraph `interrupt()`/resume 运行时（langgraph 包，不在本仓库源码内）。
- `StructuredTool` 本体（langchain-core）。

## 8. 与相邻子系统交互

- 上游用户 → 本叶子：构造 `MCPAdapter(target)`，`await list_tools()` 得 LangChain 工具。
- 本叶子 → agent-factory：把转换出的 `BaseTool` 列表传入 `create_agent(tools=...)`。
- 本叶子 → 外部 MCP 服务器：经 FastMCP client 发起 `list_tools/call_tool`。
- 本叶子 → langgraph：elicitation 用 `interrupt()` 与人在回路中间件协同（见 agent-middleware 的 HumanInTheLoopMiddleware）。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **适配器模式**：本叶子是典型"编排/适配层"——把外部 MCP 协议工具适配成 LangChain 工具契约，实际协议在 FastMCP。
- **配置驱动**：传输类型、缓存模式、elicitation 行为均由构造/调用参数决定。
- **beta 命名空间**：导入即警告，体现 API 稳定性标记。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| MCP 适配架构图 | `mcp-integration-architecture.html` | architecture | standard |
| MCP 工具调用时序（含 elicitation） | `mcp-integration-sequence.html` | sequence | standard |

JSON IR 源文件位于 `json/`。降档说明：时序涉及外部 MCP 服务器与中断分支，按 standard 档渲染。
