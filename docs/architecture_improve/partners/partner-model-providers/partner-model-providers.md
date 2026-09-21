# partner-model-providers 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文归并分析 10 家较小的模型提供商集成，深读代表（ollama/groq/mistralai/deepseek/xai/openrouter）+ 共性列表说明。
> 不重复展开 OpenAI、Anthropic、向量存储、Exa 搜索等相邻叶子。
>
> 源码基准：`libs/partners/` 下 10 个 provider 包；branch `master`，commit `4492ad7a`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| **深读代表** | | |
| ChatOllama | 本地 Ollama 服务的 Chat 模型，无 API key，HTTP 本地服务，支持 reasoning 模式 | `ollama/langchain_ollama/chat_models.py:262` |
| OllamaEmbeddings | Ollama 本地嵌入 | `ollama/langchain_ollama/embeddings.py` |
| OllamaLLM | Ollama 文本补全 | `ollama/langchain_ollama/llms.py` |
| ChatGroq | Groq 云 Chat 模型（OpenAI 兼容 API），消息转换自实现 | `groq/langchain_groq/chat_models.py:126` |
| ChatMistralAI | Mistral AI Chat 模型，自定义错误映射（httpx status code → Model*Error） | `mistralai/langchain_mistralai/chat_models.py:619` |
| MistralAIEmbeddings | Mistral 嵌入 | `mistralai/langchain_mistralai/embeddings.py` |
| ChatDeepSeek | DeepSeek Chat，**直接继承 `BaseChatOpenAI`**，复用 OpenAI 适配层，仅覆盖 base_url 与缓存 token | `deepseek/langchain_deepseek/chat_models.py:110` |
| ChatXAI | xAI（Grok）Chat，**继承 `BaseChatOpenAI`**（OpenAI 兼容封装） | `xai/langchain_xai/chat_models.py:61` |
| ChatOpenRouter | OpenRouter 多模型路由 Chat，**继承 `BaseChatModel` 自实现转换**（用 langchain_core 的 openai block_translators） | `openrouter/langchain_openrouter/chat_models.py:106` |
| **共性列表（不穷举）** | | |
| ChatPerplexity | Perplexity Chat + 搜索工具/检索器 | `perplexity/langchain_perplexity/chat_models.py:617` |
| PerplexitySearchResults / Retriever | Perplexity 搜索工具与检索器 | `perplexity/langchain_perplexity/tools.py`、`retrievers.py` |
| ChatFireworks / Fireworks / FireworksEmbeddings / FireworksRerank | Fireworks Chat + LLM + 嵌入 + 重排 | `fireworks/langchain_fireworks/chat_models.py:799` |
| ChatHuggingFace / HuggingFaceEndpoint / HuggingFacePipeline / HuggingFaceEmbeddings | HuggingFace 本地 pipeline + 端点 + 嵌入 | `huggingface/langchain_huggingface/` |
| NomicEmbeddings | Nomic 嵌入（仅嵌入，无 Chat） | `nomic/langchain_nomic/embeddings.py` |

**如实披露**：10 家中深读 ollama/groq/mistralai/deepseek/xai/openrouter 6 个代表，其余以共性列表说明，不穷举每家逐行细节。

**事实修正（相对基线）**：基线把 openrouter 归入"继承 `BaseChatOpenAI`"的一类，经源码核实 **`ChatOpenRouter` 实际继承 `BaseChatModel`**（`openrouter/.../chat_models.py:106`），自实现消息转换（复用 langchain_core 的 openai block_translators，但不继承 OpenAI 包的 `BaseChatOpenAI`）。本叶子据此修正三种继承模式的归类。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ChatOllama(BaseChatModel)` | `ollama/.../chat_models.py:262` | 本地 Ollama Chat 模型，直接继承 `BaseChatModel`（不经 OpenAI 适配层） |
| `ChatGroq(BaseChatModel)` | `groq/.../chat_models.py:126` | Groq Chat，自实现 `_convert_message_to_dict`/`_convert_dict_to_message` |
| `ChatMistralAI(BaseChatModel)` | `mistralai/.../chat_models.py:619` | Mistral Chat，自实现错误映射 |
| `_status_error_type` / `_raise_on_error` | `mistralai/.../chat_models.py:265,282` | httpx 状态码 → langchain `Model*Error` 异常类选择 |
| `ChatDeepSeek(BaseChatOpenAI)` | `deepseek/.../chat_models.py:110` | **复用 OpenAI 适配层**，仅覆盖 base_url 指向 DeepSeek 端点，附加 prompt cache token 处理 |
| `ChatXAI(BaseChatOpenAI)` | `xai/.../chat_models.py:61` | xAI，复用 OpenAI 适配层 |
| `ChatOpenRouter(BaseChatModel)` | `openrouter/.../chat_models.py:106` | 多模型路由，自实现转换（引用 langchain_core openai block_translators） |
| `ChatHuggingFace` / `HuggingFaceEndpoint` / `HuggingFacePipeline` | `huggingface/.../chat_models/`、`llms/` | HF 本地 pipeline 与推理端点两种模式 |
| `PerplexitySearchResults` / `PerplexitySearchRetriever` | `perplexity/.../tools.py`、`retrievers.py` | Perplexity 搜索工具与检索器（非纯 Chat） |
| `FireworksRerank` | `fireworks/.../rerank.py` | Fireworks 重排（非 Chat/Embeddings） |

**三种继承模式**（本叶子核心观察，已按源码修正）：
1. **直接继承 `BaseChatModel` 自实现转换**（ollama、groq、mistralai、openrouter、perplexity、fireworks）：自实现消息转换与错误映射；
2. **继承 `BaseChatOpenAI`**（deepseek、xai）：复用 OpenAI 适配层，仅覆盖 endpoint 与 API key 解析（OpenAI 兼容 API 提供商的最简集成）；
3. **本地推理**（huggingface pipeline、ollama）：不调云端 API，在本地进程内运行模型。

## 3. 关键调用链

### 链 1：ChatOllama（本地模型）

1. `ChatOllama.invoke([HumanMessage("hi")])`，框架调用 `_generate`。
2. Ollama 无 API key，默认 `http://localhost:11434`；消息经 `_convert_messages_to_ollama_messages` 转为 Ollama 原生格式。
3. 调用 `ollama` Python SDK 的 `chat()` 方法（HTTP POST 到本地 Ollama 服务）。
4. 响应经 `_convert_ollama_response_to_message` 转回 `AIMessage`。
5. reasoning 模式（`<think>` 标签）由 `reasoning` 参数控制，推理内容提取到 `additional_kwargs["reasoning_content"]`。

### 链 2：ChatDeepSeek（复用 OpenAI 适配层）

1. `ChatDeepSeek` 继承 `BaseChatOpenAI`，`_generate`/`_stream` 直接复用 OpenAI 实现。
2. 构造时把 `openai_api_base` 指向 DeepSeek 端点（`https://api.deepseek.com`）。
3. 额外在响应后处理中提取 `prompt_cache_hit_tokens` 并附加到 message metadata（`chat_models.py:49,81`）。
4. 这是"OpenAI 兼容 API 提供商"的最简集成模式——零消息转换代码。

### 链 3：ChatGroq（自实现转换）

1. `ChatGroq._generate` 调用 groq SDK 的 `client.chat.completions.create`。
2. 消息经 `_convert_message_to_dict`（`:1354`）逐条转换（OpenAI 兼容格式）。
3. 响应经 `_convert_dict_to_message`（`:1511`）转回 langchain message。
4. 错误映射：`groq.BadRequestError` → `GroqContextOverflowError`（`:93`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `model` | 模型名（各提供商不同） | 各 chat_models.py |
| `base_url` / `openai_api_base` | 本地或云端端点 | 各包 |
| API key | 各包环境变量（`OLLAMA_HOST`、`GROQ_API_KEY`、`DEEPSEEK_API_KEY` 等） | 各包 |
| `temperature` / `max_tokens` / `top_p` | 采样参数 | 各包 |
| `reasoning`（ollama） | 推理模式开关 | `ollama/.../chat_models.py:277` |
| `num_predict`（ollama） | 最大生成 token | `ollama/.../chat_models.py:293` |
| `max_retries` | 各包默认（通常 2） | 各包 |
| Ollama 特有 | `format`、`keep_alive`、`mirostat` 等 Ollama 原生参数 | `ollama/.../chat_models.py` |

## 5. 错误与重试语义

- **groq**：`GroqContextOverflowError`（`groq.BadRequestError` 子类）；`_handle_groq_invalid_request`（`:97`）。
- **mistralai**：自实现 httpx 状态码映射（`_status_error_type`，`:265`）——401→认证、403→权限、404→模型不存在、429→限流、5xx→API 错误；`_raise_on_error`（`:282`）抛对应异常。
- **deepseek/xai**：继承 `BaseChatOpenAI`，复用 OpenAI 错误映射。
- **重试**：各包透传 SDK 重试（通常 `max_retries=2`），或自实现 retry decorator（如 mistralai `_create_retry_decorator`，`:115`）。
- **ollama**：本地服务连接错误透传 `ollama` SDK 异常。

## 6. 并发细节

- **同步/异步双实现**：各包 `_generate`/`_stream` 与 `_agenerate`/`_astream` 成对。
- **客户端懒初始化**：各包在 `validate_environment` 构建 SDK 客户端。
- **无共享可变状态**：pydantic 实例字段即配置。
- **本地模型（ollama/hf pipeline）**：不引入网络线程；HF pipeline 可能在本地加载模型权重。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- 10 个 provider 包的 chat_models / embeddings / llms / tools / retrievers / rerank 适配层
- 各包的 `_compat.py`（SDK 版本兼容）与 `data/_profiles.py`（模型 profile）

**Out-of-Scope（不在本仓库源码内）**

- 各提供商 API 服务端（Groq/Mistral/DeepSeek/xAI/OpenRouter/Perplexity/Fireworks 云端）——外部服务
- Ollama 本地服务端——外部进程
- HuggingFace Hub 远端模型权重、Transformers 库——外部依赖
- 各提供商 Python SDK（groq/mistralai/ollama 等）——依赖项
- 相邻叶子：OpenAI / Anthropic / 向量存储 / Exa 搜索

## 8. 与相邻子系统交互

- **上游 → 本叶子**：用户应用通过 `BaseChatModel`/`Embeddings` 标准接口调用各 provider 类。
- **本叶子 → 下游**：
  - 云端 provider → 各自 SDK → 云端 API（外部）；
  - ollama → ollama SDK → 本地 Ollama 服务（外部进程）；
  - huggingface pipeline → 本地 Transformers 模型（外部库，权重在 HF Hub）。
- **同域交互**：deepseek/xai 复用 `langchain_openai` 的 `BaseChatOpenAI`（跨包依赖）；openrouter 复用 langchain_core 的 openai block_translators；其余包独立适配 `langchain_core`。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：本叶子展示 partner 集成的**三种继承模式**——(1) 直接继承 `BaseChatModel` 自实现转换；(2) 继承 `BaseChatOpenAI` 复用 OpenAI 适配层；(3) 本地推理不调云端 API。这是纯 Python 框架库"接口在消费方、实现在集成方"依赖倒置的三种落地方案。
- **注册表与工厂**：各包 `__init__.py` 直接导出类，无字符串→类注册表；`data/_profiles.py` 是模型能力数据（上下文窗口等），非注册表。
- **可选依赖**：各包依赖对应 SDK（groq/mistralai/ollama 等）；huggingface 用 `utils/import_utils.py` 做可选依赖检测（transformers 非硬依赖，仅在使用本地 pipeline 时需要）。
- **配置驱动**：deepseek/xai 通过继承 `BaseChatOpenAI` + 覆盖 `openai_api_base` 实现端点切换；ollama 的 `reasoning` 参数切换推理模式。
- **双轨实现**：huggingface 有 `HuggingFaceEndpoint`（调远端推理 API）与 `HuggingFacePipeline`（本地进程内推理）两种模式。
- **如实披露**：10 家中深读 6 个代表，其余以共性列表说明；每家的逐行调用链未穷举。
- **外部边界**：所有云端 API、本地服务端、SDK 均标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 模型提供商集成架构图 | `partner-model-providers-architecture.html` | architecture | standard |
| Chat 请求通用时序图 | `partner-model-providers-sequence.html` | sequence | **showcase** |

**相对基线的提升**：基线仅有 1 张 architecture（standard），本次**新增第 2 张 sequence（showcase）**，补齐叶子级 ≥2 张图配额；并在源码核实后修正了 openrouter 的基类归类（`BaseChatModel` 而非 `BaseChatOpenAI`）。架构图因跨"langchain_core ↔ 三种继承模式 ↔ 各 SDK ↔ 云端/本地服务"多层、分支较多，showcase 布局校验未一次通过，降为 standard 渲染。JSON IR 源文件位于 `json/` 目录。
