# 提示词模板（core-prompts）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「把输入变量渲染成 PromptValue 的模板体系」，不重复展开消息类型（见 `../core-messages/core-messages.md`）与输出解析（见 `../core-output-parsers/core-output-parsers.md`）。
>
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/prompts/` + `example_selectors/` + `prompt_values.py`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `BasePromptTemplate` | 所有提示词模板的基类，继承 `RunnableSerializable[dict, PromptValue]`，本身是 Runnable | `prompts/base.py:38` |
| 变量契约 | `input_variables`/`optional_variables`/`partial_variables`/`input_types` | `prompts/base.py:43-67` |
| 变量名校验 | 禁止变量名 `stop`（内部保留），model_validator 校验 | `prompts/base.py:80` |
| `PromptTemplate` | 字符串模板，支持 f-string/jinja2/mustache 格式 | `prompts/prompt.py:24` |
| 格式化器 | `jinja2_formatter`/`mustache_formatter`/`validate_f_string_template`/`check_valid_template` | `prompts/string.py:33`/`112`/`230`/`262` |
| 模板变量提取 | `get_template_variables` 按格式解析模板占位符 | `prompts/string.py:298` |
| `ChatPromptTemplate` | 聊天模板：消息模板列表，渲染成 `ChatPromptValue` | `prompts/chat.py:794` |
| 消息模板基类 | `BaseMessagePromptTemplate` | `prompts/message.py:16` |
| 角色消息模板 | `HumanMessagePromptTemplate`/`AIMessagePromptTemplate`/`SystemMessagePromptTemplate`/`ChatMessagePromptTemplate` | `prompts/chat.py:668`/`677`/`686`/`354` |
| 消息占位符 | `MessagesPlaceholder`：在模板中插入运行期 `list[BaseMessage]` | `prompts/chat.py:53` |
| 图文消息模板 | `_StringImageMessagePromptTemplate`：文本+图像多模态 | `prompts/chat.py:397` |
| 文档格式化 | `format_document`：把 `Document` 填进模板 | `prompts/base.py:452` |
| `PromptValue` 体系 | `PromptValue` ABC → `StringPromptValue`/`ChatPromptValue`/`ImagePromptValue`/`ChatPromptValueConcrete` | `prompt_values.py:24`/`54`/`80`/`135`/`152` |
| 少样本选择器 | `BaseExampleSelector` ABC；`LengthBasedExampleSelector`/`SemanticSimilarityExampleSelector`/`MaxMarginalRelevanceExampleSelector` | `example_selectors/base.py:9`、`length_based.py:18`、`semantic_similarity.py:101`/`231` |
| 少样本模板 | `FewShotPromptTemplate`/`FewShotChatMessagePromptTemplate` | `prompts/few_shot.py`、`few_shot_with_templates.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `BasePromptTemplate` | `prompts/base.py:38` | 提示词模板契约，泛型 `RunnableSerializable[dict, PromptValue]`；定义 `format`/`format_prompt` 抽象与 `partial` 绑定 |
| `StringPromptTemplate` | `prompts/string.py:328` | 产出字符串的模板 ABC |
| `PromptTemplate` | `prompts/prompt.py:24` | 单条字符串模板的具体实现 |
| `ChatPromptTemplate` | `prompts/chat.py:794` | 由若干消息模板组成，是 LCEL 中最常用的模板类型 |
| `BaseMessagePromptTemplate` | `prompts/message.py:16` | 单条消息模板基类 |
| `MessagesPlaceholder` | `prompts/chat.py:53` | 占位符，运行期替换为外部消息列表（用于历史/多轮） |
| `PromptValue` | `prompt_values.py:24` | 模板渲染结果的 ABC；`to_messages()`/`to_string()` |
| `BaseExampleSelector` | `example_selectors/base.py:9` | 少样本示例选择器 ABC（`select_examples`） |
| `SemanticSimilarityExampleSelector` | `example_selectors/semantic_similarity.py:101` | 基于向量库相似度选示例（依赖 `VectorStore`，外部实现） |
| `jinja2_formatter`/`mustache_formatter` | `prompts/string.py:33`/`112` | 可插拔模板语法格式化器 |

## 3. 关键调用链

**调用链一：`prompt.invoke(vars)` 渲染（`BasePromptTemplate` 作为 Runnable）**

1. 用户调 `chat_prompt.invoke({"question": "..."})`，因 `BasePromptTemplate` 是 `RunnableSerializable`，走标准 Runnable 执行链（见 `core-runnables`）。
2. 子类 `_format_prompt`/`format_prompt` 把 `partial_variables` + 用户传入变量合并，按模板格式（f-string/jinja2/mustache）填充。
3. `ChatPromptTemplate` 遍历消息模板列表：普通消息模板填变量成 `HumanMessage` 等；`MessagesPlaceholder` 把外部消息列表直接插入。
4. 产出 `ChatPromptValue`，其 `to_messages()` 给出 `list[BaseMessage]` 传给 chat model。

**调用链二：少样本选择（`FewShotPromptTemplate` + selector）**

1. 调用时先经 `BaseExampleSelector.select_examples(input_variables)` 选出若干示例。
2. 把示例填进示例模板，拼到主模板前面，再渲染最终提示词。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `template_format` | `f-string`/`jinja2`/`mustache`，选择占位符语法 | `prompts/string.py` |
| `input_variables` | 必填变量名列表，缺变量运行期报错 | `base.py:43` |
| `partial_variables` | 预先绑定的变量，调用时无需再传 | `base.py:67` |
| `output_parser` | 模板自带的输出解析器（与 LLM 输出配对） | `base.py:64` |
| `selector.k` | 语义相似度选择器返回示例数 | `semantic_similarity.py` |

## 5. 错误与重试语义

- **变量缺失**：渲染时若 `input_variables` 未全部提供且无 partial 兜底，抛模板变量错误。
- **变量名 `stop`**：构造时 model_validator 直接 `ValueError`（`base.py:83`），因 `stop` 是 LLM 停止序列内部参数。
- **模板语法校验**：`check_valid_template`/`validate_jinja2` 在构造期校验模板可解析，提前发现语法错误。
- **无网络/无重试**：本叶子是纯渲染层；语义相似度选择器才触及向量库（外部），本叶子不负责重试。

## 6. 并发细节

- **Runnable 即并发**：因模板是 `RunnableSerializable`，并发/batch/stream 能力全部由 `core-runnables` 基类提供；模板本身无状态、线程安全。
- **partial 绑定**：`partial_variables` 是不可变映射，多协程共享同一模板实例安全。
- **无共享锁**：渲染是纯函数式，无临界区。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `prompts/` 全部模板与格式化器、`example_selectors/`、`prompt_values.py`。

**Out-of-Scope（不在本仓库源码内）**
- 向量库实现（供 `SemanticSimilarityExampleSelector`）在 `core-vectorstores-retrievers`/partners；本叶子只定义 `VectorStore` 接口。
- 实际 LLM 调用在 `core-language-models`/partners；本叶子只产 `PromptValue`。
- jinja2/mustache 是第三方模板引擎（作为依赖安装，不在本仓库源码内）。

## 8. 与相邻子系统交互

- **上游（用户 / Runnable 链）**：模板通常是 LCEL 链的第一环（`prompt | model | parser`）。
- **本叶子 → 模型（`core-language-models`）**：`PromptValue.to_messages()` 产出 `list[BaseMessage]` 喂给 chat model。
- **本叶子 → 输出解析（`core-output-parsers`）**：`BasePromptTemplate.output_parser` 可自带解析器，模板与解析配对。
- **本叶子 → 向量库（`core-vectorstores-retrievers`）**：语义相似度示例选择器依赖 `VectorStore`。
- **本叶子 → 消息（`core-messages`）**：`ChatPromptValue.to_messages()` 产出 `BaseMessage` 列表。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BasePromptTemplate`/`BaseMessagePromptTemplate`/`PromptValue`/`BaseExampleSelector` 是抽象基类契约；模板格式（f-string/jinja2/mustache）是可插拔格式化钩子链。
- **Runnable 化**：模板直接继承 `RunnableSerializable`，因此天然支持 `|` 组合、invoke/batch/stream——这是「契约即组合」能力缝的典型体现。
- **配置驱动**：`template_format` 字符串选择不同格式化器，`partial_variables`/`input_variables` 驱动渲染行为。
- **注册表**：格式化器按 `template_format` 字符串分发；少样本选择器是策略对象注入。
- **图类型侧重**：用 architecture 表达模板类层级与渲染管道；无循环状态机，不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 提示词模板类层级与渲染管道架构图 | `core-prompts-architecture.html` | architecture | standard（showcase 对「中心 hub 多向扇出 + 跨层连线」布局约束严格，降 standard） |

JSON IR 源文件位于 `json/core-prompts-architecture.json`。本叶子为模板渲染层，不补 sequence/dataflow 图。
