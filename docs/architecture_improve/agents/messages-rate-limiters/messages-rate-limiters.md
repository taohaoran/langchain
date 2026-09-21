# 消息工具与速率限制器（messages-rate-limiters）

> 本文是 `agents` 域下的叶子子系统文档。域级总览见 `../agents.md`，本文只展开 v1 包面对消息类型与
> 速率限制器的符号聚合；这些类型的本体在 langchain-core。
>
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，源码目录 `libs/langchain_v1/langchain/messages/` 与
> `libs/langchain_v1/langchain/rate_limiters/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 角色消息类型 | 再导出 `HumanMessage / AIMessage / SystemMessage / ToolMessage / AnyMessage` | `messages/__init__.py:11,16,35,31,19` |
| 消息分块 | `AIMessageChunk / ToolCallChunk / ServerToolCallChunk` | `messages/__init__.py:9,34,28` |
| 内容块类型 | `TextContentBlock / ImageContentBlock / AudioContentBlock / VideoContentBlock / FileContentBlock / DataContentBlock / ReasoningContentBlock / PlainTextContentBlock / Citation` | `messages/__init__.py` |
| 工具调用结构 | `ToolCall / InvalidToolCall / ServerToolCall / ServerToolResult / RemoveMessage` | `messages/__init__.py:33,20,29,30,27` |
| 用量元数据 | `UsageMetadata / InputTokenDetails / OutputTokenDetails` | `messages/__init__.py:36,19,23` |
| 消息裁剪 | `trim_messages`，按 token/策略裁剪历史 | `messages/__init__.py:38` |
| 速率限制器基类 | `BaseRateLimiter`，与 `BaseChatModel` 配合限流 | `rate_limiters/__init__.py:8` |
| 内存速率限制器 | `InMemoryRateLimiter` 进程内实现 | `rate_limiters/__init__.py:8` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `AnyMessage` | 来自 langchain_core | 消息联合类型别名 |
| `trim_messages` | 来自 langchain_core | 历史消息裁剪工具 |
| `BaseRateLimiter` | 来自 langchain_core | 限流抽象基类 |
| `InMemoryRateLimiter` | 来自 langchain_core | 进程内令牌桶式限流实现 |

## 3. 关键调用链

本叶子不实现逻辑，仅聚合符号。典型用法：

1. 用户 `from langchain.messages import HumanMessage, AIMessage` 构造对话消息列表。
2. 消息列表经 `create_agent` 传入，模型调用前由工厂前置 `SystemMessage`（见 agent-factory 叶子）。
3. `BaseRateLimiter` 实例传给具体 `BaseChatModel`，由模型在请求前 `acquire` 限流。

## 4. 配置项

- 本叶子无独立配置项；消息字段（content、tool_calls、usage_metadata 等）与速率限制器参数（请求间隔等）均定义于 langchain-core。

## 5. 错误与重试语义

- 本叶子不处理错误；`InvalidToolCall` 表示模型产出了非法工具调用，供上层校验。

## 6. 并发细节

- `InMemoryRateLimiter` 为进程内实现，其内部并发控制在 langchain-core 内（不在本仓库源码内）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- v1 包面对消息类型与速率限制器的统一符号聚合。

**Out-of-Scope（不在本仓库源码内）**
- 消息类型、内容块、`trim_messages`、`BaseRateLimiter / InMemoryRateLimiter` 本体均在 langchain-core，不在本叶子源码内。

## 8. 与相邻子系统交互

- 上游 agent-factory / 中间件 → 本叶子：使用 `HumanMessage/AIMessage/ToolMessage` 等类型构造与处理状态。
- 本叶子 → 下游 langchain-core：纯再导出，无额外逻辑。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **前端符号面工程**：本叶子是 v1 包面的符号聚合，把 langchain-core 的消息与限流符号在 v1 命名空间统一导出，提升导入一致性。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| 消息与限流符号聚合架构图 | `messages-rate-limiters-architecture.html` | architecture | standard |
| 请求期消息裁剪与限流数据流 | `messages-rate-limiters-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/`。本轮新增 dataflow 图：本叶子两类符号在请求期构成一条"原始历史消息 → `trim_messages` 按 token/策略裁剪 → 窗口化上下文 → `InMemoryRateLimiter.acquire` 取令牌 → 放行模型调用"的数据管道，把静态符号聚合落到真实运行数据流。
