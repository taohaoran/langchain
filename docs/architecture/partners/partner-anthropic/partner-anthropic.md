# partner-anthropic 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文只展开 Anthropic Claude 集成（`langchain-anthropic`）的职责边界，不重复展开其他相邻叶子。
>
> 源码基准：`libs/partners/anthropic/`，`langchain-anthropic` 包。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Chat 模型封装 | `ChatAnthropic` 封装 Anthropic Messages API，支持同步/异步、流式/批量 | `langchain_anthropic/chat_models.py:1058` |
| 消息格式化 | `_format_messages` 把 langchain messages 转为 Anthropic API 的 system + messages 二元组 | `chat_models.py:539` |
| 消息合并 | `_merge_messages` 合并连续同角色消息（Anthropic API 要求交替 user/assistant） | `chat_models.py:326` |
| 多模态内容块转换 | image_url → image source、data content block、tool_use block、reasoning block | `chat_models.py:234,390,516` |
| 工具调用转换 | langchain `ToolCall` ↔ Anthropic `tool_use` content block；`convert_to_anthropic_tool` | `chat_models.py:2831,2926` |
| 工具绑定 | `bind_tools` 支持 tool_choice / parallel_tool_calls / strict | `chat_models.py:2321` |
| 结构化输出 | `with_structured_output` 支持 function_calling / json_schema 模式 | `chat_models.py:2526` |
| Thinking 推理模式 | `thinking` 参数与 reasoning_effort；thinking 开启时结构化输出的降级处理 | `chat_models.py:2292` |
| Beta 特性路由 | `betas` 字段走 `client.beta.messages.create` | `chat_models.py:1155` |
| 文本补全 LLM（遗留） | `AnthropicLLM` 封装旧版 text completion | `langchain_anthropic/llms.py:137` |
| 文件工具中间件 | State/Filesystem 两种后端的 Claude 文件编辑与记忆工具 | `middleware/anthropic_tools.py` |
| Bash 工具中间件 | `ClaudeBashToolMiddleware` 封装 shell 工具 | `middleware/bash.py:19` |
| 文件搜索中间件 | `StateFileSearchMiddleware` 基于 include pattern 的文件搜索 | `middleware/file_search.py:89` |
| Prompt 缓存中间件 | `AnthropicPromptCachingMiddleware` 自动注入 `cache_control` 断点 | `middleware/prompt_caching.py:47` |
| 输出解析器 | 重导出 langchain_core 的 Anthropic 工具解析器 | `output_parsers.py` |
| SDK 兼容 | 按 anthropic SDK 版本做兼容适配 | `_compat.py`、`_sdk_compat.py` |
| 模型 profile 数据 | 内置 Claude 模型能力 profile | `data/_profiles.py` |

对外导出面（`langchain_anthropic/__init__.py`）：`ChatAnthropic`、`AnthropicLLM`、`convert_to_anthropic_tool`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ChatAnthropic(BaseChatModel)` | `chat_models.py:1058` | Claude Chat 模型主类，定义 `model`/`max_tokens`/`temperature`/`betas` 等配置；实现 `_generate`/`_stream`/`_agenerate`/`_astream` |
| `_format_messages` | `chat_models.py:539` | **核心适配层**：把 `list[BaseMessage]` 转为 `(system, messages)` 二元组——Anthropic API 的 system 是独立顶层参数，不在 messages 数组内 |
| `_merge_messages` | `chat_models.py:326` | 合并连续同角色消息，满足 Anthropic API 的 user/assistant 交替要求 |
| `_format_image` | `chat_models.py:234` | image_url → Anthropic `image` source block |
| `_format_data_content_block` | `chat_models.py:390` | data content block → Anthropic 原生 block |
| `_format_text_block` | `chat_models.py:516` | 文本块格式化 |
| `_lc_tool_calls_to_anthropic_tool_use_blocks` | `chat_models.py:2926` | langchain `ToolCall` → Anthropic `tool_use` content block |
| `convert_to_anthropic_tool` | `chat_models.py:2831` | langchain 工具定义 → `AnthropicTool` TypedDict |
| `AnthropicTool(TypedDict)` | `chat_models.py:139` | Anthropic 工具定义类型（name/description/input_schema） |
| `_AnthropicCommon(BaseLanguageModel)` | `llms.py:27` | 遗留 LLM 公共基类 |
| `AnthropicLLM(LLM, _AnthropicCommon)` | `llms.py:137` | 旧版文本补全 |
| `AnthropicPromptCachingMiddleware` | `middleware/prompt_caching.py:47` | 继承 `AgentMiddleware`，给 system message 与 tools 自动打 `cache_control` |
| `ClaudeBashToolMiddleware` | `middleware/bash.py:19` | 继承 `ShellToolMiddleware`，封装 bash 工具 |
| `StateClaudeTextEditorMiddleware` / `FilesystemClaudeTextEditorMiddleware` | `middleware/anthropic_tools.py:599,1090` | Claude 文件编辑工具的两种状态后端 |
| `StateFileSearchMiddleware` | `middleware/file_search.py:89` | 文件搜索工具中间件 |
| 错误映射族 | `chat_models.py:952-992` | anthropic SDK 异常 → langchain_core `Model*Error`（含 `AnthropicOverloadedError`） |

**依赖倒置**：接口在 `langchain_core`（`BaseChatModel`、`AgentMiddleware`、`LLM`），实现在本包。

## 3. 关键调用链

### 链 1：Chat 补全请求

1. 用户调用 `model.invoke([HumanMessage("hi")])`，框架层调用 `ChatAnthropic._generate`（`chat_models.py:2250`）。
2. `_generate` 调用 `_get_request_payload`：内部用 `_format_messages(messages)`（`:539`）把消息列表转为 `(system, formatted_messages)` 二元组：
   - SystemMessage 被提取为顶层 `system` 参数（Anthropic API 特有，不放在 messages 数组里）；
   - 其余消息经 `_merge_messages`（`:326`）合并连续同角色后，逐条按 block 类型格式化（image/data/tool_use/tool_result/text）；
   - `tool_use` block 与 AIMessage.tool_calls 做去重合并（`:600-617`）。
3. payload 合并默认参数后，`self._create(payload)` 调用 anthropic SDK 的 `client.messages.create`（或 `client.beta.messages.create` 当 `betas` 非空）。
4. 异常分支：`anthropic.BadRequestError` → `_handle_anthropic_bad_request`（`:1010`）；`anthropic.APIError` → `_handle_anthropic_api_error`（`:1024`）。
5. 成功后 `_format_output(response, ...)`（`:2267`）把 Anthropic 响应（content blocks 数组）转回 langchain `ChatResult`，其中 `tool_use` block → `AIMessage.tool_calls`，`text` block → content。

### 链 2：流式补全

1. `_stream`（`:1862`）构建相同 payload 但带 `stream=True`。
2. 逐 chunk 迭代 Anthropic SSE 事件（message_start/content_block_start/content_block_delta/content_block_stop/message_delta/message_stop）。
3. 每个 delta 经格式化函数转为 `ChatGenerationChunk` 并 `yield`。
4. 异步路径 `_astream`（`:1912`）对称实现。

### 链 3：Prompt 缓存中间件

1. `AnthropicPromptCachingMiddleware.before_model` 在调用模型前，遍历 messages 与 tools。
2. `_tag_system_message`（`:206`）与 `_tag_tools`（`:246`）给最后一个符合条件的 block 打 `{"cache_control": {"type": "ephemeral"}}`。
3. Anthropic API 检测到 `cache_control` 后缓存前缀，降低后续请求 token 成本。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `model`（alias `model_name`） | 必填，如 `"claude-sonnet-4-5-20250929"` | `chat_models.py:1095` |
| `max_tokens`（alias `max_tokens_to_sample`） | `None`，从模型 profile 的 `max_output_tokens` 动态推断 | `chat_models.py:1098` |
| `temperature` / `top_k` / `top_p` | `None` | `chat_models.py:1108,1111,1114` |
| `max_retries` | `2`（anthropic SDK 默认） | `chat_models.py:1121` |
| `default_request_timeout`（alias `timeout`） | `None` | `chat_models.py:1117` |
| `anthropic_api_key`（alias `api_key`） | 从 `ANTHROPIC_API_KEY` 环境变量读取 | `chat_models.py:1136` |
| `anthropic_api_url`（alias `base_url`） | 解析顺序：kwarg → `ANTHROPIC_API_URL` → `ANTHROPIC_BASE_URL` → LangSmith Gateway | `chat_models.py:1127` |
| `anthropic_proxy` | 从 `ANTHROPIC_PROXY` 读取 | `chat_models.py:1143` |
| `betas` | `None`；非空时走 `client.beta.messages.create` | `chat_models.py:1155` |
| `thinking` | 控制推理模式；开启时结构化输出有降级警告 | `chat_models.py:2292` |
| `stop_sequences`（alias `stop`） | `None` | `chat_models.py:1124` |

## 5. 错误与重试语义

- **异常映射**（`chat_models.py:952-992`）：
  - `anthropic.BadRequestError` → `AnthropicContextOverflowError` / `AnthropicInvalidRequestError`；
  - `anthropic.AuthenticationError` → `AnthropicAuthenticationError`；
  - `anthropic.NotFoundError` → `AnthropicModelNotFoundError`；
  - `anthropic.RateLimitError` → `AnthropicRateLimitError`；
  - `anthropic.OverloadedError` → `AnthropicOverloadedError`（Anthropic 特有的过载错误）；
  - `anthropic.InternalServerError` → `AnthropicAPIError`；
  - `anthropic.APIConnectionError` / `APITimeoutError` → 连接/超时错误。
- **重试**：`max_retries=2` 默认，由 anthropic SDK 客户端负责重试。
- **thinking 模式与结构化输出冲突**：`_get_llm_for_structured_output_when_thinking_is_enabled`（`:2292`）发出警告，若模型未产生 tool call 则抛 `OutputParserException`。

## 6. 并发细节

- **同步/异步双实现**：`_generate`/`_stream` 与 `_agenerate`/`_astream` 对称。
- **客户端懒初始化**：`client`/`async_client` 在 `validate_environment` 构建。
- **无共享可变状态**：pydantic 实例字段即配置。
- **中间件钩子**：`AgentMiddleware` 的 `before_model`/`after_model` 在模型调用前后同步执行，不引入额外线程。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `langchain_anthropic/chat_models.py`：Chat 模型适配、消息格式化、工具转换
- `langchain_anthropic/llms.py`：遗留文本补全
- `langchain_anthropic/middleware/`：文件工具、Bash、文件搜索、prompt 缓存中间件
- `langchain_anthropic/output_parsers.py`：解析器重导出
- `langchain_anthropic/data/`：模型 profile

**Out-of-Scope（不在本仓库源码内）**

- Anthropic Claude API 服务端（messages API、beta API）——外部服务
- `anthropic` Python SDK（依赖项）
- Claude 工具执行的运行时环境（文件系统、shell）——由中间件包装但执行在用户进程
- 相邻叶子：OpenAI / 向量存储 / 其他模型提供商 / Exa 搜索

## 8. 与相邻子系统交互

- **上游 → 本叶子**：用户应用通过 `BaseChatModel` 接口调用 `ChatAnthropic`；中间件可由 LangGraph agent 装配。
- **本叶子 → 下游**：
  - `ChatAnthropic._generate` → `anthropic.Anthropic` SDK → Anthropic API（外部）；
  - 中间件操作用户进程的文件系统/shell（`Filesystem*` 后端）或内存状态（`State*` 后端）。
- **同域交互**：依赖 `langchain_core` 的 `BaseChatModel`/`AgentMiddleware`/`AgentState` 抽象；与其他 partner 包无直接代码依赖。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：核心能力缝是 **`_format_messages` 的 system/messages 分离**——Anthropic API 把 system 作为独立顶层参数（而非 messages 数组中的一条 SystemMessage），这与 OpenAI 把 system 当 messages[0] 的做法根本不同；以及 **content block 数组模型**（text/image/tool_use/tool_result/reasoning 都是同层 block）。
- **消息合并**：`_merge_messages` 满足 Anthropic API 的 user/assistant 严格交替要求，这是 Anthropic 特有的适配逻辑。
- **注册表与工厂**：`convert_to_anthropic_tool` 是工厂函数；中间件通过 `__init__.py` 直接导出类，用户显式装配到 agent。
- **可选依赖**：`anthropic` SDK 是硬依赖；`_sdk_compat.py` 处理 SDK 版本差异。
- **配置驱动**：`betas` 非空时路由到 `client.beta.messages.create`；`thinking` 开启时结构化输出走降级路径——同一 `_generate` 代码按配置分派。
- **外部边界**：Anthropic API 服务端、`anthropic` SDK 标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Anthropic 集成架构图 | `partner-anthropic-architecture.html` | architecture | standard |
| 消息格式化时序图 | `partner-anthropic-sequence.html` | sequence | standard |

降档说明：架构图跨"langchain_core ↔ ChatAnthropic 适配层 ↔ anthropic SDK ↔ Anthropic API"四层，showcase 布局校验未一次通过，降为 standard；时序图因参与者框宽度限制降为 standard。JSON IR 源文件位于 `json/` 目录。
