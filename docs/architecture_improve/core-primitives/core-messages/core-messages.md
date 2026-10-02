# 消息类型体系（core-messages）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「LLM 交互的通用消息数据结构」，不重复展开可组合执行单元（见 `../core-runnables/core-runnables.md`）与聊天模型（见 `../core-language-models/core-language-models.md`）。
>
> 源码基准：`langchain-core` master，commit `89252a8f7043a74f2300729fd038df8221a87e1e`，源码位于 `libs/core/langchain_core/messages/`（11 个 `.py` 文件 + `content.py` + `block_translators/`，约 6200 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `BaseMessage` 抽象基类 | 聊天模型输入/输出的通用数据结构，pydantic `Serializable` | `messages/base.py:93` |
| 标准消息字段 | `content`、`additional_kwargs`、`response_metadata`、`type`、`name`、`id` | `messages/base.py:103-140` |
| 类型化内容块 | `content_blocks` 属性把 content 解析为类型化 `ContentBlock`（text/tool_call/tool_call_chunk/citation 等） | `messages/base.py:200`、`messages/content.py` |
| `BaseMessageChunk` | 消息分片基类，`__add__` 把流式分片合并成完整消息 | `messages/base.py:409` |
| 人类消息 | `HumanMessage` + `HumanMessageChunk` | `messages/human.py:9`、`63` |
| AI 消息 | `AIMessage` + `AIMessageChunk`；含 `UsageMetadata`/`input_token_details`/`output_token_details` | `messages/ai.py:160`、`418`、`104` |
| 系统消息 | `SystemMessage` + `SystemMessageChunk` | `messages/system.py:9`、`63` |
| 工具消息 | `ToolMessage`（带 `ToolOutputMixin`）+ `ToolMessageChunk`；含 `ToolCall`/`ToolCallChunk` TypedDict | `messages/tool.py:26`、`174`、`206` |
| 函数消息 | `FunctionMessage` + `FunctionMessageChunk`（历史函数调用 API） | `messages/function.py:15`、`34` |
| 任意角色消息 | `ChatMessage`（任意 role）+ `ChatMessageChunk` | `messages/chat.py:15`、`25` |
| 删除指令 | `RemoveMessage`：在消息列表里标记删除某条历史消息 | `messages/modifier.py:8` |
| 提供者格式转换 | `block_translators/`：把 OpenAI/Anthropic/Bedrock Converse/Google GenAI/v0 格式统一转成 v1 `ContentBlock` | `messages/block_translators/`、`base.py:207-221` |
| 序列化往返 | `messages_from_dict`/`_message_from_dict`/`_create_message_from_message_type`：dict↔消息 | `messages/utils.py:547`、`515`、`593` |
| 灵活输入转换 | `convert_to_messages`：把 str/BaseMessage/tuple/tool_calls 等统一转成 `list[BaseMessage]` | `messages/utils.py:786`、`706` |
| 缓冲区字符串 | `get_buffer_string`：把消息列表拼成可读文本 | `messages/utils.py:287` |
| 消息工具 | `filter_messages`（按类型/id 过滤）、`merge_message_runs`（合并连续同角色）、`trim_messages`（按 token 数裁剪历史） | `messages/utils.py:857`、`1002`、`1133` |
| OpenAI 适配 | `convert_to_openai_messages`/`_get_message_openai_role`/`_convert_to_openai_tool_calls` | `messages/utils.py:1530`、`2209`、`2230` |
| 分片↔完整 | `message_chunk_to_message`/`_msg_to_chunk`/`_chunk_to_msg` | `messages/utils.py:560`、`2152`、`2168` |
| 近似计数 | `count_tokens_approximately` | `messages/utils.py:2244` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `BaseMessage` | `messages/base.py:93` | 全部消息的基类；`type` 字段用于反序列化时识别子类；pydantic `extra="allow"` 容忍提供者私有字段 |
| `BaseMessageChunk` | `messages/base.py:409` | 继承 `BaseMessage`，重载 `__add__` 实现流式分片累加 |
| `AIMessage` / `AIMessageChunk` | `messages/ai.py:160`/`418` | 模型输出；携带 `usage_metadata`（token 用量）与 tool_calls |
| `UsageMetadata` | `messages/ai.py:104` | `input_tokens`/`output_tokens`/`total_tokens` TypedDict |
| `ToolCall` / `ToolCallChunk` | `messages/tool.py:206`/`content.py:247` | 模型发起的工具调用（name/args/id）及流式分片 |
| `HumanMessage`/`SystemMessage`/`ToolMessage`/`FunctionMessage`/`ChatMessage` | 各自文件 | 按角色区分的消息子类，各自配 Chunk 变体 |
| `RemoveMessage` | `messages/modifier.py:8` | 非真实消息，是「从历史删除某 id 消息」的指令对象 |
| `TextContentBlock`/`Citation`/`NonStandardAnnotation` | `messages/content.py:207`/`126`/`184` | 类型化内容块 TypedDict，统一多模态/引用/工具调用的标准表示 |
| `convert_to_messages` | `messages/utils.py:786` | 归一化入口：把任意「消息样表示」转成强类型 `list[BaseMessage]` |
| `trim_messages` | `messages/utils.py:1133` | 按 token 预算裁剪历史消息列表（保留系统消息等策略可配） |
| `merge_message_runs` | `messages/utils.py:1002` | 合并连续同角色消息为一条 |
| `_convert_to_openai_messages` | `messages/utils.py:1530` | 把消息列表转成 OpenAI chat completions 格式（`convert_to_openai_messages` 的重载分发） |

## 3. 关键调用链

**调用链一：把用户输入归一化为消息列表（`convert_to_messages`，`utils.py:786`）**

1. 接收 `MessageLikeRepresentation`（可能是 str、tuple、dict、`BaseMessage`、或其列表）。
2. 逐条经 `_convert_to_message`（`utils.py:706`）：已是 `BaseMessage` 直接返回；dict/tuple 经 `_create_message_from_message_type`（`utils.py:593`）按 `type` 字段构造对应子类；str 包成 `HumanMessage`。
3. 返回 `list[BaseMessage]`，供 chat model 或 prompt 消费。

**调用链二：流式分片合并（`BaseMessageChunk.__add__`，`base.py:409`）**

1. 模型流式产出 `AIMessageChunk`，每个 chunk 含增量 content/tool_call_delta。
2. 逐 chunk 调用 `chunk = chunk + next_chunk`，`__add__` 把文本拼接、tool_calls 按 index 合并、usage 累加。
3. 流结束后 `message_chunk_to_message`（`utils.py:560`）把累积 chunk 转成完整 `AIMessage`。

**调用链三：内容块归一化（`BaseMessage.content_blocks`，`base.py:200`）**

1. 访问 `message.content_blocks` 时，按需延迟 import 各 `block_translators`（openai/anthropic/bedrock/google/v0）。
2. 按 content 的实际格式分发到对应转换器，统一产出 v1 `ContentBlock` 列表（text/tool_call/citation 等）。
3. 延迟 import 是为避免循环依赖（`base.py:206` 注释明确说明）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `BaseMessage.type` | 子类各自固定字符串（`human`/`ai`/`system`/`tool` 等），反序列化识别用 | `base.py:117` |
| `content_blocks` | 访问时才懒解析，不存字段 | `base.py:200` |
| `trim_messages` 策略参数 | `max_tokens`、`strategy`(last/first)、`token_counter`、`include_system`、`allow_partial` 等 | `utils.py:1133` |
| `model_config.extra="allow"` | pydantic 容忍提供者私有额外字段 | `base.py:142` |
| `id` 字段 | 由 provider 提供的消息唯一 id，`coerce_numbers_to_str` | `base.py:135` |

## 5. 错误与重试语义

- **反序列化容错**：`_message_from_dict`/`_create_message_from_message_type` 按 `type` 字符串路由；未知类型走默认或报错，类型字符串来自 `_get_message_type_str`（`utils.py:247`）。
- **分片合并**：`__add__` 假设 chunk 来自同一流；tool_call 按 `index` 对齐，缺失 index 时按出现顺序追加。
- **trim_messages**：当单条消息就超 token 预算时，按 `allow_partial` 决定是否截断文本或直接保留整条；`_first_max_tokens`/`_last_max_tokens`（`utils.py:1970`/`2086`）分别处理从头/从尾保留策略。
- **无网络/无重试**：本叶子是纯数据结构层，不发起调用，无重试；错误是类型/数据错误，立即抛出。

## 6. 并发细节

- **无共享状态**：消息是不可变-ish 的值对象，并发安全来自「每消息独立」；唯一可变点是流式 chunk 的 `__add__` 产生新对象而非原地修改。
- **pydantic 模型**：`BaseMessage` 是 pydantic `Serializable` 子类，字段校验在构造时完成；线程间传递无需锁。
- **延迟 import**：`content_blocks` 属性在访问时才 import `block_translators`，规避循环依赖；不涉及线程安全问题。
- **无异步原语**：本叶子全为同步纯函数/数据类；async 支持由消费方（模型/Runnable）提供。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `messages/` 全部：消息基类、各角色消息及 Chunk 变体、类型化内容块、block_translators、序列化/裁剪/合并/过滤工具。

**Out-of-Scope（不在本仓库源码内）**
- 实际 LLM API 调用（OpenAI/Anthropic 等）在 `libs/partners/*`；本叶子只定义其输入输出数据结构。
- 多模态底层编解码（图像 base64、音频）依赖外部库，本叶子只做结构表示。
- 聊天历史持久化存储（`BaseChatMessageHistory`）见 `core-documents-loaders`；prompt 模板见 `core-prompts`。

## 8. 与相邻子系统交互

- **上游（模型/Runnable）**：`BaseChatModel`/`LLM` 接收 `list[BaseMessage]` 作为输入，产出 `AIMessage`/`AIMessageChunk`（见 `core-language-models`）。
- **本叶子 → 提示词（`core-prompts`）**：`BaseMessage.__add__`（`base.py:294`）支持 `message + message` 拼成 `ChatPromptTemplate`，连接消息与提示词层。
- **本叶子 → 工具（`core-tools`）**：`AIMessage.tool_calls` 触发工具调用，`ToolMessage` 承载工具执行结果回传给模型。
- **本叶子 → 历史（`core-documents-loaders`）**：`BaseChatMessageHistory` 存取 `list[BaseMessage]`。
- **本叶子 → 序列化（`core-serialization-cache-infra`）**：`Serializable` 基类提供 `to_json`/`lc_kwargs`，`messages_from_dict` 负责反序列化。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：本叶子是「抽象基类体系」能力缝——`BaseMessage` ABC 在消费方定义字段契约，各角色子类是实现方；`type` 字段 + `_create_message_from_message_type` 构成字符串→类的注册表式反序列化路由。
- **双轨实现（Chunk 双轨）**：每种消息同时有完整类（`AIMessage`）与分片类（`AIMessageChunk`），`__add__` 运行期合并——这是「流式路径 + 完整路径」双轨，与 transformers Fast/Slow 同理。
- **配置驱动**：消息内容的实际结构（纯字符串 vs 多模态 block list vs 各 provider dict 格式）由 `content` 字段内容驱动，`content_blocks` 在访问时经 block_translators 统一转换，是数据驱动的静态分派。
- **可选依赖与懒加载**：`content_blocks` 延迟 import 各 provider 转换器以规避循环依赖；不强制安装所有 provider。
- **注册表**：`_get_message_type_str`/`_create_message_from_message_type` 按 `type` 字符串分发到子类，是小型字符串→类注册表。
- **图类型侧重**：用 architecture 表达消息类型层级与归一化管道，用 dataflow 表达各 provider 格式经 block_translators 归一化为标准 ContentBlock 的数据管道；本叶子无循环状态机，不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 消息类型层级与归一化管道架构图 | `core-messages-architecture.html` | architecture | **showcase** |
| 消息内容块归一化数据流图 | `core-messages-dataflow.html` | dataflow | standard（本轮新增；4 个 provider 扇入翻译器的边标签间距差 showcase 阈值 0.7–4px，属扇入紧凑，降 standard；主管道清晰） |

JSON IR 源文件位于 `json/` 目录。本轮相对基线的改进：在基线 1 张架构图基础上，新增 1 张 dataflow 图，表达「各 provider 原始格式 → block_translators 懒解析 → 标准 ContentBlock → BaseMessage」的数据归一化管道（符合 dataflow「数据从哪来、经过什么加工、变成什么」的核心语义）。两图均实际渲染成功（退出码 0、HTML 非空）。
