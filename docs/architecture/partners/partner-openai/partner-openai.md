# partner-openai 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文只展开 OpenAI 集成（`langchain-openai`）的职责边界，不重复展开 Anthropic、向量存储、
> 其他模型提供商、Exa 搜索等相邻叶子（分别见各自叶子）。
>
> 源码基准：`libs/partners/openai/`，`langchain-openai` 包。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Chat 模型封装 | `ChatOpenAI` 封装 OpenAI Chat Completions 与 Responses 双 API，支持同步/异步、流式/批量 | `langchain_openai/chat_models/base.py:715`（`BaseChatOpenAI`）、`:2823`（`ChatOpenAI`） |
| Azure OpenAI Chat | `AzureChatOpenAI` 继承 `BaseChatOpenAI`，针对 Azure 部署做 endpoint/api_version/deployment 适配 | `langchain_openai/chat_models/azure.py:37` |
| Codex OAuth 实验模型 | `_ChatOpenAICodex` 包装 `ChatOpenAI`，以 ChatGPT 订阅 OAuth 访问 codex 后端 | `langchain_openai/chat_models/codex.py` |
| ChatGPT OAuth 令牌提供 | 刷新感知的 `Authorization` / `ChatGPT-Account-Id` 头注入 | `langchain_openai/chatgpt_oauth.py` |
| Embeddings | `OpenAIEmbeddings` 实现 `Embeddings` 接口，含 tiktoken 长度安全分块与加权平均 | `langchain_openai/embeddings/base.py:86` |
| Azure Embeddings | `AzureOpenAIEmbeddings` 适配 Azure 嵌入端点 | `langchain_openai/embeddings/azure.py` |
| 文本补全 LLM（遗留） | `OpenAI` / `AzureOpenAI` 封装旧版 text completion API | `langchain_openai/llms/base.py:57`（`BaseOpenAI`）、`:787`（`OpenAI`） |
| 消息双向转换 | `_convert_message_to_dict` / `_convert_dict_to_message` / `_convert_delta_to_message_chunk` | `chat_models/base.py:217,404,486` |
| 工具调用转换 | langchain `ToolCall` ↔ OpenAI `tool_calls` 格式互转 | `chat_models/base.py:4158`（`_lc_tool_call_to_openai_tool_call`） |
| 结构化输出 | `with_structured_output` 支持 Pydantic / JSON schema / function calling 三模式 | `chat_models/base.py:2522` |
| 工具绑定 | `bind_tools` 把 langchain 工具转 OpenAI tool 格式 | `chat_models/base.py:2413` |
| 自定义工具装饰器 | `@custom_tool` 支持自由格式字符串输入与 CFG 语法（Lark） | `langchain_openai/tools/custom_tool.py:27` |
| 输出解析器重导出 | 重导出 langchain_core 的 OpenAI 工具解析器 | `langchain_openai/output_parsers/tools.py` |
| 内容审核中间件 | `OpenAIModerationMiddleware` 在模型调用前后审核输入/输出/工具消息 | `langchain_openai/middleware/openai_moderation.py:49` |
| SDK 版本兼容 | 按安装的 `openai` SDK 版本自动选择 `httpx` 或 `httpx2` | `langchain_openai/_compat.py`、`chat_models/_compat.py` |
| 模型 profile 数据 | 内置模型能力 profile（上下文窗口、能力标志） | `langchain_openai/data/_profiles.py` |
| HTTP 客户端工具 | 代理、超时、SSL、分块超时、API key 解析 | `chat_models/_client_utils.py` |

对外导出面（`langchain_openai/__init__.py`）：`ChatOpenAI`、`AzureChatOpenAI`、`OpenAIEmbeddings`、`AzureOpenAIEmbeddings`、`OpenAI`、`AzureOpenAI`、`custom_tool`、`StreamChunkTimeoutError`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `BaseChatOpenAI(BaseChatModel)` | `chat_models/base.py:715` | OpenAI Chat 模型基类，定义 client 字段、model_name、temperature 等配置；实现 `_generate`/`_stream`/`_agenerate`/`_astream` |
| `ChatOpenAI(BaseChatOpenAI)` | `chat_models/base.py:2823` | 具体 OpenAI 官方 API 封装，覆写 `_stream`/`_astream`/`with_structured_output` 等 |
| `AzureChatOpenAI(BaseChatOpenAI)` | `chat_models/azure.py:37` | Azure 适配子类，处理 `azure_deployment`/`api_version`/`azure_ad_token` 等 Azure 特有字段 |
| `_ChatOpenAICodex(ChatOpenAI)` | `chat_models/codex.py` | 实验性 OAuth 子类，注入 codex 后端头 |
| `OpenAIEmbeddings(BaseModel, Embeddings)` | `embeddings/base.py:86` | 实现 langchain_core `Embeddings` 抽象基类的 `embed_documents`/`embed_query` |
| `BaseOpenAI(BaseLLM)` | `llms/base.py:57` | 旧版文本补全 LLM 基类 |
| `_convert_message_to_dict` | `chat_models/base.py:404` | **核心适配层**：langchain `BaseMessage` → OpenAI API dict（按消息类型分派 role/content/tool_calls） |
| `_convert_dict_to_message` | `chat_models/base.py:217` | 反向：OpenAI 响应 dict → langchain `BaseMessage` |
| `_convert_delta_to_message_chunk` | `chat_models/base.py:486` | 流式增量 → langchain `BaseMessageChunk` |
| `_convert_message_to_dict` 中的 role 分派 | `chat_models/base.py:416-482` | HumanMessage→user、AIMessage→assistant、SystemMessage→system、ToolMessage→tool、FunctionMessage→function |
| `OpenAIModerationMiddleware` | `middleware/openai_moderation.py:49` | 继承 `AgentMiddleware`，`before_model`/`after_model` 钩子做内容审核 |
| `custom_tool` 装饰器 | `tools/custom_tool.py:27` | 包装普通函数为 langchain `BaseTool`，metadata 标记 `{"type": "custom_tool"}` |
| 错误映射族 | `chat_models/base.py:570-608` | 将 openai SDK 异常映射为 langchain_core 的 `Model*Error` 体系（认证/限流/上下文溢出等） |

**依赖倒置**：接口定义在消费方 `langchain_core`（`BaseChatModel`、`Embeddings`、`BaseLLM`、`AgentMiddleware`），实现在本包。这是 partners 域的通用模式。

## 3. 关键调用链

### 链 1：Chat 补全请求（`ChatOpenAI.invoke` → OpenAI API）

1. 用户调用 `model.invoke([HumanMessage("hi")])`，langchain_core `BaseChatModel` 框架层调用子类 `_generate`（`chat_models/base.py:1852`）。
2. `_generate` 调用 `_get_request_payload`（`:1939`）：
   - `self._convert_input(input_).to_messages()` 把输入归一化为 `list[BaseMessage]`；
   - 合并 `_default_params()` 与 kwargs 得 payload；
   - 若 `_use_responses_api(payload)` 为真，走 `_construct_responses_api_payload` 构建 Responses API 入参；
   - 否则走 Chat Completions：逐条 `_convert_message_to_dict(m)`（`:404`）把 `BaseMessage` 转为 `{"role":..., "content":...}` dict。
3. 根据 payload 选择 API 端点：
   - 有 `response_format` → `root_client.chat.completions.with_raw_response.parse(**payload)`（`:1867`）；
   - 走 Responses API → `root_client.responses.with_raw_response.create/parse`（`:1873-1879`），再用 `_construct_lc_result_from_responses_api`（`:5015`）转换结果；
   - 默认 → `self.client.with_raw_response.create(**payload)`（`:1904`）。
4. 异常分支：`openai.BadRequestError` → `_handle_openai_bad_request`（`:612`）；`openai.APIError` → `_handle_openai_api_error`（`:649`），分别映射为 `Model*Error` 异常族。
5. 成功后 `_create_chat_result(response, generation_info)`（`:1970`）把 OpenAI 响应 dict 转为 langchain `ChatResult`（含 `ChatGeneration` 列表与 `AIMessage`）。

### 链 2：流式补全

1. `_stream`（`:1775`）构建与 `_generate` 相同的 payload，但带 `"stream": True`。
2. 同步路径：`self.client.with_streaming_response.create(**payload)` 迭代 chunk。
3. 每个 chunk 经 `_convert_delta_to_message_chunk`（`:486`）转为 `ChatGenerationChunk` 并 `yield`。
4. 异步路径 `_astream`（`:2067`）与 `_astream_responses`（`:1682`）对称实现。
5. `_client_utils.py` 的 `_astream_with_chunk_timeout` 提供分块级超时保护。

### 链 3：Embedding 长文本安全分块

1. `embed_documents`（`embeddings/base.py:725`）按 `chunk_size` 分批。
2. 若 `check_embedding_ctx_length=True`，走 `_get_len_safe_embeddings`（`:577`）：用 tiktoken 分词，按 token 上限（`MAX_TOKENS_PER_REQUEST=300000`，`:22`）切分。
3. 多块文本经 `_process_batched_chunked_embeddings`（`:26`）按 token 数加权平均后归一化。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `model_name`（alias `model`） | `"gpt-3.5-turbo"` | `chat_models/base.py:733` |
| `openai_api_key`（alias `api_key`） | 从 `OPENAI_API_KEY` 环境变量推断；支持 str / sync callable / async callable | `chat_models/base.py:742` |
| `openai_api_base`（alias `base_url`） | 解析顺序：显式 kwarg → `OPENAI_API_BASE` → `OPENAI_BASE_URL` | `chat_models/base.py:794` |
| `temperature` / `top_p` / `max_tokens` / `n` | `None`（由 API 默认决定） | `chat_models/base.py:736,869,872,875` |
| `max_retries` | `None`（由 openai SDK 默认重试） | `chat_models/base.py:841` |
| `request_timeout`（alias `timeout`） | `None` | `chat_models/base.py:816` |
| `stream_usage` | 默认开启；若设置了自定义 `base_url` 或自定义 client 则关闭 | `chat_models/base.py:824` |
| `reasoning_effort` | Chat Completions 推理模型用：`minimal`/`low`/`medium`/`high` | `chat_models/base.py:878` |
| `reasoning` / `verbosity` | Responses API 推理模型用 | `chat_models/base.py:909,924` |
| `use_responses_api` | `bool` 或自动推断（`output_version=="responses/v1"`、`reasoning` 非空等触发） | `chat_models/base.py:1924` |
| `chunk_size`（embeddings） | 类默认；`MAX_TOKENS_PER_REQUEST=300000` | `embeddings/base.py:22` |
| `check_embedding_ctx_length` | 是否启用 tiktoken 长度安全分块 | `embeddings/base.py:743` |
| `azure_deployment` / `api_version` | Azure 特有 | `chat_models/azure.py:37` |
| `openai_proxy` | 从 `OPENAI_PROXY` 环境变量读取 | `chat_models/base.py:812` |

## 5. 错误与重试语义

- **异常映射**：openai SDK 异常被映射为 langchain_core `Model*Error` 体系（`chat_models/base.py:570-608`）：
  - `openai.BadRequestError` → `OpenAIAPIContextOverflowError` / `OpenAIInvalidRequestError`；
  - `openai.AuthenticationError` → `OpenAIAuthenticationError`；
  - `openai.RateLimitError` → `OpenAIRateLimitError`；
  - `openai.NotFoundError` → `OpenAIModelNotFoundError`；
  - `openai.InternalServerError` → `OpenAIAPIError`；
  - `openai.APIConnectionError` / `APITimeoutError` → 连接/超时错误。
- **重试**：重试由底层 `openai` SDK 客户端负责（`max_retries` 字段透传），本包不实现应用层重试队列。
- **上下文溢出**：`_handle_openai_bad_request`（`:612`）检测上下文溢出错误并抛出 `ContextOverflowError` 子类，供上层做自动截断/重试。
- **流式分块超时**：`_client_utils.py` 的 `_astream_with_chunk_timeout` 在两次 chunk 之间设置超时，超时抛 `StreamChunkTimeoutError`。
- **审核中间件失败**：`OpenAIModerationMiddleware` 命中违规时抛 `OpenAIModerationError`（`middleware/openai_moderation.py:24`）。

## 6. 并发细节

- **同步/异步双实现**：`_generate`/`_stream` 与 `_agenerate`/`_astream` 对称成对实现，分别走 `self.client` 与 `self.async_client`。
- **客户端懒初始化**：`client`/`async_client` 字段默认 `None`，在 `validate_environment`/`_ensure_sync_client_available` 时构建；`root_client`/`root_async_client` 持有带 raw_response 能力的根客户端。
- **Codex 令牌获取**：异步路径通过 `aget_token` 从事件循环外获取 OAuth 令牌，经私有 kwarg `_codex_headers` 传递给同步 payload 构建器，避免在线程池中重复获取跨进程文件锁（`chat_models/codex.py:58-66`）。
- **Embeddings 并发**：`aembed_documents` 使用 `run_in_executor` 包装同步客户端调用。
- **无共享可变状态**：pydantic 模型实例字段即配置，无全局可变缓存（除 `_ssrf_client` 单例与 `_experimental_warning_emitted` 布尔）。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `langchain_openai/chat_models/`：Chat 模型双 API 适配、消息转换、工具调用转换、结构化输出
- `langchain_openai/embeddings/`：Embeddings 接口实现与长度安全分块
- `langchain_openai/llms/`：遗留文本补全 LLM
- `langchain_openai/tools/`：自定义工具装饰器
- `langchain_openai/middleware/`：内容审核中间件
- `langchain_openai/output_parsers/`：解析器重导出
- `langchain_openai/data/`：模型 profile 数据

**Out-of-Scope（不在本仓库源码内）**

- OpenAI 官方 API 服务端（chat/completions、responses、embeddings、moderation 等端点）——外部服务
- Azure OpenAI 服务端部署——外部服务
- `openai` Python SDK 本身（依赖项）
- `tiktoken` 分词库（依赖项）
- `httpx` / `httpx2` 传输库（依赖项）
- ChatGPT Codex 后端（`chatgpt.com/backend-api/codex`）——非官方实验端点
- 相邻叶子：Anthropic / 向量存储 / 其他模型提供商 / Exa 搜索

## 8. 与相邻子系统交互

- **上游 → 本叶子**：用户应用或 langchain 链（`RunnableSequence`、`create_react_agent`）通过 langchain_core `BaseChatModel` 接口调用 `ChatOpenAI`；调用方只依赖 `langchain_core` 抽象，不感知 OpenAI 具体格式。
- **本叶子 → 下游**：
  - `ChatOpenAI._generate` → `openai.OpenAI` SDK 客户端 → OpenAI API 服务端（外部）；
  - `OpenAIEmbeddings.embed_documents` → `openai` embeddings 端点（外部）；
  - `OpenAIModerationMiddleware` → OpenAI moderation 端点（外部）。
- **同域交互**：本包依赖 `langchain_core` 的 `BaseChatModel`/`Embeddings`/`BaseLLM`/`BaseTool`/`AgentMiddleware` 抽象基类与 `convert_to_openai_tool` 等工具函数；其他 partner 包（anthropic 等）与本包无直接代码依赖，均独立适配 `langchain_core`。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：本叶子是典型的"接口在消费方、实现在集成方"依赖倒置——`langchain_core` 定义 `BaseChatModel`/`Embeddings`/`BaseLLM` 抽象基类，本包提供 OpenAI 具体实现。核心能力缝是**消息转换层**（`_convert_message_to_dict` ↔ `_convert_dict_to_message`）与**工具调用转换**（langchain `ToolCall` ↔ OpenAI `tool_calls`）。
- **注册表与工厂**：本包不维护字符串→类注册表；通过 `__init__.py` 直接导出具体类，由用户显式实例化。`bind_tools`/`with_structured_output` 是工厂式方法，把 langchain 工具/schema 转为 OpenAI 格式绑定到模型实例。
- **可选依赖与懒加载**：`openai` SDK 是硬依赖（`pyproject.toml` 声明）；`_compat.py` 按 SDK 版本动态选择 `httpx`/`httpx2`；`client` 字段懒初始化到首次调用。
- **配置驱动**：运行时行为（走 Chat Completions 还是 Responses API）由 `use_responses_api` 字段与 `_model_prefers_responses_api(model_name)` 推断共同决定，同一套 `_generate` 代码路径按配置分派到不同 API 构建函数。
- **双轨实现**：Chat Completions API 与 Responses API 是同一 `_generate` 方法下的两条实现轨（`_construct_responses_api_payload` vs `_convert_message_to_dict`），按 payload/配置自动选择。
- **外部边界**：OpenAI/Azure 服务端、`openai` SDK、`tiktoken` 均标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| OpenAI 集成架构图 | `partner-openai-architecture.html` | architecture | standard |
| Chat 请求时序图 | `partner-openai-sequence.html` | sequence | standard |

降档说明：架构图跨"langchain_core 抽象层 ↔ OpenAI 适配层 ↔ openai SDK ↔ 外部服务"四层，组件数较多，showcase 布局校验未一次通过，降为 standard 渲染；时序图因参与者框宽度与子标签长度受限，showcase 校验未通过，降为 standard。JSON IR 源文件位于 `json/` 目录。
