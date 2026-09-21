# 聊天模型与嵌入封装层（chat-models-embeddings）

> 本文是 `agents` 域下的叶子子系统文档。域级总览见 `../agents.md`，本文只展开 v1 的模型初始化工厂与
> 可配置模型封装；具体聊天模型实现在各 partner 包（不在本叶子源码内）。
>
> 源码基准：`langchain_v1` master，commit `4492ad7a8`，源码目录 `libs/langchain_v1/langchain/chat_models/` 与
> `libs/langchain_v1/langchain/embeddings/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `init_chat_model` | 字符串 `provider:model` 或 `(model, model_provider=...)` 解析为具体 `BaseChatModel` | `chat_models/base.py:195` |
| 模型字符串解析 | 拆 `provider:model` 前缀，归一化 provider 名（`-`→`_`、小写） | `chat_models/base.py:619` |
| provider 推断 | 按模型名前缀（gpt-/claude/gemini/grok/sonar...）推断 provider | `chat_models/base.py:543` |
| 惰性导入 | `_get_chat_model_creator` 按 provider 惰性 import partner 包模型类 | `chat_models/base.py:151,123` |
| `_ConfigurableModel` | 延迟到调用期按 `RunnableConfig` 构造的可配置模型 Runnable | `chat_models/base.py:657` |
| 声明式方法转发 | `bind_tools / with_structured_output` 等延迟方法转发到真实模型 | `chat_models/base.py:654,681` |
| `init_embeddings` | 嵌入模型的字符串初始化工厂 | `embeddings/base.py:191` |
| 嵌入 provider 解析 | `_parse_model_string / _infer_model_and_provider` | `embeddings/base.py:103,158` |
| 对外导出 | `BaseChatModel / init_chat_model`、`Embeddings / init_embeddings` | `chat_models/__init__.py:3`；`embeddings/__init__.py:11` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `init_chat_model(...)` | `chat_models/base.py:195` | 模型工厂主入口 |
| `_parse_model(model, provider)` | `chat_models/base.py:619` | 解析与推断 provider |
| `_get_chat_model_creator(provider)` | `chat_models/base.py:151` | 返回惰性构造函数 |
| `_ConfigurableModel(Runnable)` | `chat_models/base.py:657` | 可配置模型，延迟构造 |
| `init_embeddings(...)` | `embeddings/base.py:191` | 嵌入模型工厂 |

## 3. 关键调用链

**调用链：`init_chat_model("openai:gpt-4o")`**

1. 进入 `init_chat_model`（`chat_models/base.py:195`）。
2. `_parse_model`：检测 `:` 前缀且 `openai` 在 `_BUILTIN_PROVIDERS` → 拆出 provider 与模型名（`chat_models/base.py:622`）。
3. provider 归一化为小写下划线（`chat_models/base.py:646`）。
4. `_get_chat_model_creator("openai")` 惰性 import `langchain_openai` 并取得 `ChatOpenAI` 构造函数（`chat_models/base.py:539`）。
5. 调用构造函数 `ChatOpenAI(model="gpt-4o", **kwargs)` 返回实例。
6. 若未显式 provider 且无 `:`，走 `_attempt_infer_model_provider` 按前缀推断；推断失败抛 `ValueError` 并列支持的 provider（`chat_models/base.py:631`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|------|------|
| `model` | 必填，字符串或已实例化模型 | `chat_models/base.py:195` |
| `model_provider` | None，自动从字符串前缀或模型名推断 | `chat_models/base.py:535` |
| `**kwargs` | 透传给具体模型构造（temperature、api_key 等） | `chat_models/base.py:536` |
| Gemini 前缀 | `gemini` 默认推断为 `google_vertexai` 并发 `DeprecationWarning` | `chat_models/base.py:580` |

## 5. 错误与重试语义

- **无法推断 provider**：抛 `ValueError`，列出支持的 provider 与文档链接（`chat_models/base.py:633`）。
- **provider 包缺失**：惰性 import 失败时给出安装提示（`_import_module`）。
- **无重试语义**：本层只做工厂解析，不承担调用重试（重试由 ModelRetryMiddleware 负责）。

## 6. 并发细节

- **惰性构造**：`_ConfigurableModel` 把模型实例化延迟到首次 `invoke`/`_model`，支持运行期经 `with_config` 切换模型。
- **无线程同步**：工厂期为纯函数解析，具体模型实例的并发由各 partner 实现负责。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `init_chat_model / init_embeddings` 工厂、provider 解析与推断、可配置模型封装。

**Out-of-Scope（不在本仓库源码内）**
- 具体聊天模型实现（`langchain_openai / langchain_anthropic` 等 partner 包，本 monorepo `libs/partners/` 或外部仓）。
- `BaseChatModel / Embeddings` 基类本体（langchain-core）。
- 旧版社区嵌入与 `CacheBackedEmbeddings` 已迁至 `langchain-classic`（不在本叶子源码内，见 `embeddings/__init__.py:3` 警告）。

## 8. 与相邻子系统交互

- 上游 agent-factory → 本叶子：`create_agent(model="openai:...")` 调 `init_chat_model` 实例化模型。
- 本叶子 → 下游 partner：经惰性 import 加载 `libs/partners/` 下的具体模型类。
- 本叶子 → 下游用户：返回 `BaseChatModel` 供直接调用或传入 `create_agent`。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **注册表与工厂**：本叶子是"字符串→类"路由工厂，`_BUILTIN_PROVIDERS` + 惰性 import 构成注册表；与 v1 中间件钩子注册同为框架级扩展机制。
- **配置驱动**：同一工厂经不同 provider/kwargs 构造不同模型实例。
- **懒加载工程**：惰性 import partner 包，避免顶层 import 全部模型依赖。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| 模型初始化工厂架构图 | `chat-models-embeddings-architecture.html` | architecture | standard |

JSON IR 源文件位于 `json/`。本叶子为工厂路由，单张架构图即可；时序图不适用故省略。
