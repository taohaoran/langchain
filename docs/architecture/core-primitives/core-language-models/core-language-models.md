# 语言模型抽象（core-language-models）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「聊天模型/补全模型/交叉编码器的抽象基类」，不重复展开消息（见 `../core-messages/core-messages.md`）与 Runnable（见 `../core-runnables/core-runnables.md`）。
>
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/language_models/` + `cross_encoders.py`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `BaseLanguageModel` | 所有语言模型的基类，继承 `RunnableSerializable` | `language_models/base.py:181` |
| prompt 生成 | `generate_prompt`：接收 PromptValue 产出 LLMResult | `base.py:318` |
| token 计数 | `get_num_tokens`/`get_num_tokens_from_messages`/`get_token_ids` | `base.py:448`/`465`/`434` |
| 结构化输出 | `with_structured_output`：约束模型输出到 schema | `base.py:405` |
| LangSmith 参数 | `_get_ls_params`：上报到 LangSmith 的模型标识 | `base.py:421` |
| 版本注入 | `model_post_init` 自动记录 langchain 版本 | `base.py:222` |
| `BaseChatModel` | 聊天模型基类，泛型 `BaseLanguageModel[AIMessage]` | `chat_models.py:284` |
| 聊天生成 | 抽象 `_generate`/`_stream`/`_agenerate`/`_astream` | `chat_models.py` |
| invoke/stream | 非抽象 `invoke`/`stream`：把输入转 messages，调 `_generate` | `chat_models.py:475`/`727` |
| 流式升级 | `generate_from_stream`/`_chat_model_stream_v3`：把分片聚合成完整结果 | `chat_models.py:218`/`995` |
| 模型 profile | `_resolve_model_profile`/`ModelProfile`：模型能力元数据 | `chat_models.py:398`、`model_profile.py` |
| 补全模型 | `LLM`：legacy 文本补全模型 | `language_models/llms.py` |
| 交叉编码器 | `BaseCrossEncoder.score(text_pairs)`：文本对相似度打分 | `cross_encoders.py:8` |
| 测试替身 | `FakeListChatModel`/`GenericFakeChatModel`：单元测试用假模型 | `fake_chat_models.py`、`fake.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `BaseLanguageModel` | `language_models/base.py:181` | 模型契约根；本身是 Runnable，`invoke`/`batch`/`stream` 由 Runnable 提供 |
| `BaseChatModel` | `chat_models.py:284` | 聊天模型契约；子类实现 `_generate`/`_stream` |
| `LLM` | `llms.py` | 文本补全模型（非聊天）抽象 |
| `BaseCrossEncoder` | `cross_encoders.py:8` | 交叉编码器 ABC：`score(list[(str,str)]) -> list[float]` |
| `ModelProfile` | `model_profile.py` | 模型能力元数据（上下文窗口、支持特性） |
| `LangSmithParams` | `base.py:43` | 上报 LangSmith 的模型标识 TypedDict |

## 3. 关键调用链

**调用链一：`chat_model.invoke(messages)`（`BaseChatModel.invoke`，`chat_models.py:475`）**

1. `_convert_input`（`chat_models.py:461`）把 `LanguageModelInput`（PromptValue/str/list）转成 `list[BaseMessage]`。
2. 走 Runnable 执行链，`_generate(messages, stop, run_manager)` 被子类实现（实际调 OpenAI/Anthropic API）。
3. 产出 `ChatResult`（含 `ChatGeneration` 列表）；`invoke` 取首条 `AIMessage` 返回。
4. 全程经回调 `on_chat_model_start`/`on_llm_new_token`/`on_llm_end`（见 core-callbacks-tracers）。

**调用链二：流式输出（`stream`，`chat_models.py:727`）**

1. `_should_stream` 判断是否走流式协议。
2. 子类 `_stream` 产出 `ChatGenerationChunk` 流；`generate_from_stream`（`chat_models.py:218`）可把分片聚合成完整 `ChatResult`。
3. `_chat_model_stream_v3`（`chat_models.py:995`）处理新版流式事件协议。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `model_name`/`model` | 模型标识 | 子类 |
| `temperature`/`max_tokens`/`stop` | 采样与停止参数，经 `**kwargs` 传给底层 API | `_generate` |
| `default_headers`/`timeout`/`max_retries` | HTTP 调用配置 | 子类（partners） |
| `ModelProfile` | 模型能力（上下文窗口等） | `model_profile.py` |
| `verbose` | 单模型详细日志 | `base.py:292` |

## 5. 错误与重试语义

- **模型错误归一化**：`_generate_response_from_error`（`chat_models.py:112`）把 API 错误转成 `ChatGeneration` 错误响应。
- **重试**：HTTP 层重试（max_retries）在 partners 实现；本叶子定义契约。
- **流式错误**：流式中途出错经 `on_llm_error` 回调上报。
- **无业务重试**：本叶子是抽象层；重试策略由 `with_retry`（core-runnables）或 partners 客户端负责。

## 6. 并发细节

- **Runnable 化**：模型是 `RunnableSerializable`，并发/batch/stream 由 `core-runnables` 基类提供；`abatch` 默认并行调 `ainvoke`。
- **异步原生**：`BaseChatModel` 的 `_agenerate`/`_astream` 是原生 async，IO 密集型适合并发。
- **无共享锁**：模型实例在单次调用间无状态（除缓存）；多协程共享实例安全。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `language_models/` 全部抽象基类、token 计数、profile、假模型；`cross_encoders.py`。

**Out-of-Scope（不在本仓库源码内）**
- 实际 API 客户端（OpenAI/Anthropic/Ollama）在 `libs/partners/*`；本叶子只定义 `_generate` 契约。
- tokenizer 是第三方库（tiktoken 等），`get_tokenizer` 延迟加载。
- 模型权重与推理服务不在本仓库源码内。

## 8. 与相邻子系统交互

- **上游（提示词/Runnable 链）**：`prompt | chat_model`；接收 PromptValue/messages。
- **本叶子 → 消息（`core-messages`）**：输入 `list[BaseMessage]`，输出 `AIMessage`/`AIMessageChunk`。
- **本叶子 → 回调（`core-callbacks-tracers`）**：`on_chat_model_start`/`on_llm_new_token`/`on_llm_end` 是流式与追踪注入点。
- **本叶子 → 输出解析（`core-output-parsers`）**：`with_structured_output` 把模型输出接到 parser。
- **本叶子 → 工具（`core-tools`）**：模型产出 `tool_calls` 触发工具循环。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseLanguageModel`/`BaseChatModel`/`LLM`/`BaseCrossEncoder` 是抽象基类契约，定义在消费方；实现方在 partners（依赖倒置）。
- **Runnable 化**：模型直接是 Runnable，天然接入 LCEL 组合。
- **双轨（sync/async/stream）**：`_generate`/`_agenerate`/`_stream`/`_astream` 四原语成对，流式分片聚合成完整结果。
- **配置驱动**：采样参数、ModelProfile 驱动模型行为；`_should_stream` 按配置决定是否走流式协议。
- **图类型侧重**：architecture 表达模型抽象层级与「messages→_generate→AIMessage」管道。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 语言模型抽象层级与生成管道架构图 | `core-language-models-architecture.html` | architecture | showcase |

JSON IR 源文件位于 `json/core-language-models-architecture.json`。本叶子不补 sequence/dataflow 图。
