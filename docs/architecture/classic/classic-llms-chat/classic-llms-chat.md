# 经典 LLM 与聊天模型（classic-llms-chat）

> 本文是 `classic` 域下的叶子子系统文档。域级总览见 `../classic.md`，本文只展开经典包的
> **语言模型基类兼容层与内置实现转发**，不展开 Chain（见 `../classic-chains/`）与提示词
> （见 `../classic-utils-eval/`）。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `4492ad7a8`。

## 1. 功能清单

经典包在本叶子提供语言模型的**旧导入路径兼容层**：真正的抽象基类已下沉到
`langchain-core`，真正的各家厂商实现已迁移到 `langchain-community` 或独立 partner 包。
本叶子源码位于 `llms/`（83 文件）、`chat_models/`（35 文件）、`base_language.py`。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 语言模型基类重导出 | `BaseLanguageModel` 仅 7 行，直接 re-export 自 core | `base_language.py:5` |
| LLM 基类重导出 | `LLM`/`BaseLLM` 19 行 re-export 自 core | `llms/base.py` |
| 文本补全模型实现转发 | ~83 个厂商文件，经 `create_importer` 动态转发到 `langchain_community.llms` | `llms/openai.py:12` 等 |
| `load_llm` 反序列化 | 按类型字符串从 `get_type_to_cls_dict` 注册表加载 | `llms/__init__.py:639` |
| 聊天模型基类遗留实现 | `_ConfigurableModel` 等兼容辅助逻辑 | `chat_models/base.py:666` |
| 聊天模型实现转发 | ~35 个厂商文件转发到 `langchain_community.chat_models` | `chat_models/openai.py:11` |
| 可配置模型包装 | `_ConfigurableModel`：运行期按提示词切换底层模型 | `chat_models/base.py:666` |

代表厂商转发：OpenAI、Anthropic、Cohere、Bedrock、VertexAI、Ollama、Azure OpenAI、
HuggingFace 等（`llms/` 与 `chat_models/` 目录清单）。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `BaseLanguageModel` | `base_language.py:5`（转发自 `langchain_core.language_models`） | 所有模型的顶层抽象，定义 `generate`/`agenerate`/`stream` 契约 |
| `BaseLLM` / `LLM` | `llms/base.py`（转发自 core） | 文本补全模型抽象，`_generate(prompts)` 契约 |
| `BaseChatModel` | `chat_models/base.py:5`（转发自 core） | 聊天模型抽象，`_generate(messages)` 契约 |
| `create_importer` / `__getattr__` | `llms/openai.py:20` | 延迟导入桩：访问属性时才从 `langchain_community` 加载并告警 |
| `get_type_to_cls_dict` | `llms/__init__.py:639` | 类型字符串 → LLM 类的注册表工厂（供 `load_llm`） |
| `_ConfigurableModel` | `chat_models/base.py:666` | Runnable 包装：运行期按配置选择底层聊天模型 |

## 3. 关键调用链

**旧路径导入（以 `from langchain_classic.llms import OpenAI` 为例）**：

1. 访问 `llms/openai.py` 的 `OpenAI` 属性 → 触发模块级 `__getattr__`
   （`llms/openai.py:20`）。
2. `create_importer` 返回的 `_import_attribute` 查 `DEPRECATED_LOOKUP` 表，定位
   `langchain_community.llms.OpenAI`。
3. 动态 `importlib.import_module` 加载 `langchain_community`，抛出
   `LangChainDeprecationWarning` 引导迁移，再返回真实类。

**`load_llm` 反序列化**：调 `get_type_to_cls_dict()`（`llms/__init__.py:639`）得到
类型→类映射，按配置 `_type` 实例化；这与 `load_chain`/`load_agent` 是同一套注册表范式。

**模型调用**：`LLMChain.generate_prompt`（见 `classic-chains/`）→
`BaseLLM.generate_prompt`（core）→ 子类 `_generate`；聊天模型走
`BaseChatModel._generate`。本叶子不承载实际推理。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `DEPRECATED_LOOKUP` | 每厂商文件内的字符串→社区模块映射表 | 各 `llms/<vendor>.py`、`chat_models/<vendor>.py` |
| `type_to_cls_dict` | `load_llm` 的类型→类注册表 | `llms/__init__.py:547` |
| 模型构造参数 | 由各厂商类（在 community/partner 包内）定义，本叶子不定义 | 不在本仓库源码内 |

## 5. 错误与重试语义

- **缺失可选依赖**：访问被转发类时若 `langchain_community` 未安装，延迟导入抛出带
  安装指引的 `ImportError`（由 `create_importer` 统一处理）。
- **未知类型**：`load_llm` 查不到 `_type` 时抛错。
- **弃用告警**：旧路径导入触发 `LangChainDeprecationWarning`，不阻断运行。
- 本层不做模型调用重试/退避，交由真实实现（community/partner）负责。

## 6. 并发细节

本叶子为静态转发层，**无自建线程/锁/协程**。真正的并发（同步 `_generate` vs
异步 `_agenerate`、流式 `_stream`）由 `langchain_core` 的 `BaseChatModel` 与各厂商
实现定义；`_ConfigurableModel` 作为 Runnable 透传调用，不引入新并发模型。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `base_language.py`、`llms/base.py`、`chat_models/base.py` 的兼容与转发逻辑；
  `llms/`、`chat_models/` 的厂商导入桩与注册表。

**Out-of-Scope（不在本仓库源码内）**
- 真正的模型抽象基类：`langchain-core`（`libs/core/`，同仓库但另一包）。
- 厂商实现：`langchain-community` 与独立 partner 包（`langchain-openai`、
  `langchain-anthropic` 等），**不在本仓库源码内**。
- 远端模型 API 调用本身。

## 8. 与相邻子系统交互

- 上游：`LLMChain`（`classic-chains/`）、Agent（`classic-agents/`）→ 持有模型 Runnable。
- 本叶子 → 下游：`langchain_core` 模型基类 → `langchain-community`/partner 实现 →
  远端 API。
- 序列化：`load_llm`（`llms/__init__.py`）与 `load_chain`（`classic-chains/`）共用
  注册表范式。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseLLM`/`BaseChatModel` 是模型能力缝，定义 `_generate`/`_stream`
  契约；本叶子只做契约的旧路径转发，不新增能力缝。
- **注册表与工厂**：`type_to_cls_dict` / `get_type_to_cls_dict` 是 `load_llm` 的
  字符串→类注册表。
- **可选依赖与懒加载**：`create_importer` + `DEPRECATED_LOOKUP` 是典型"访问时才
  import"的延迟桩，顶层 import 不强制加载 `langchain-community`。
- **目录即实例库**：`llms/` 83 文件、`chat_models/` 35 文件是"厂商目录即实例库"，
  绝大多数文件是同构转发桩，深读 1 个代表（OpenAI）即可推断全部，其余以共性列表说明。
- **与 core/v1 的关系**：这是本叶子最核心的定位——基类已下沉 core、实现已迁出，
  本包仅保留旧导入路径以兼容历史代码，不承载新功能。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 模型兼容层架构图 | `classic-llms-chat-architecture.html` | architecture | showcase |

降档披露：本叶子组件少、边界清晰（转发层 → core → community/partner → 远端），保持
`showcase`。时序图/dataflow 不适用（本叶子无自有执行流程，仅做导入转发），故不单出。
JSON IR 位于 `json/`。
