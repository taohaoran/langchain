# 输出解析器（core-output-parsers）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「把模型原始文本/Generation 解析成结构化对象」的体系，不重复展开提示词（见 `../core-prompts/core-prompts.md`）与工具（见 `../core-tools/core-tools.md`）。
>
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/output_parsers/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `BaseLLMOutputParser` | 解析器根 ABC，定义 `parse_result(list[Generation])` | `output_parsers/base.py:34` |
| `BaseGenerationOutputParser` | 把 `Generation` 对象解析为 T 的中间基类 | `output_parsers/base.py:74` |
| `BaseOutputParser` | 通用解析器基类（Runnable），`parse(text)` + `get_format_instructions()` | `output_parsers/base.py:140` |
| 流式增量解析 | `BaseTransformOutputParser`/`BaseCumulativeTransformOutputParser`：`_transform`/`_diff` 支持边流式边解析 | `output_parsers/transform.py` |
| JSON 解析 | `JsonOutputParser`：把文本解析成 JSON dict | `output_parsers/json.py:31` |
| Pydantic 解析 | `PydanticOutputParser`：按 pydantic 模型 schema 解析并校验 | `output_parsers/pydantic.py:19` |
| 列表解析 | `ListOutputParser`/`CommaSeparatedListOutputParser` | `output_parsers/list.py:43`/`139` |
| XML 解析 | `XMLOutputParser`：按 XML 标签解析 | `output_parsers/xml.py` |
| 结构化输出 | `StructuredOutputParser`：多字段指令串 | `output_parsers/structured.py` |
| 字符串解析 | `StrOutputParser`：恒等输出字符串 | `output_parsers/string.py` |
| OpenAI 函数调用解析 | `OpenAIFunctionsOutputParser`/`OpenAIToolsOutputParser` | `output_parsers/openai_functions.py`、`openai_tools.py` |
| 格式指令 | 各解析器自报 `get_format_instructions()`，拼进 prompt 引导模型 | `base.py:334` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `BaseLLMOutputParser[T]` | `output_parsers/base.py:34` | 解析器契约根；`parse_result` 接收 `list[Generation]` |
| `BaseOutputParser[T]` | `output_parsers/base.py:140` | 把文本解析为 T 的 Runnable；抽象 `parse`；`parse_with_prompt` 用 prompt 上下文解析 |
| `BaseCumulativeTransformOutputParser` | `output_parsers/transform.py` | 流式增量解析基类：`_diff` 计算相邻两次解析差量，实现边收边解 |
| `PydanticOutputParser` | `output_parsers/pydantic.py:19` | 按 pydantic model 解析；`get_format_instructions` 输出 JSON schema 指令 |
| `JsonOutputParser` | `output_parsers/json.py:31` | JSON 文本解析，支持流式累积 |
| `XMLOutputParser` | `output_parsers/xml.py` | XML 标签解析 |
| `CommaSeparatedListOutputParser` | `output_parsers/list.py:139` | 逗号分隔字符串→列表 |

## 3. 关键调用链

**调用链一：`chain = prompt | model | parser` 末端解析**

1. 模型产出 `AIMessage` 或 `list[Generation]`。
2. `parser.invoke(model_output)`：`BaseOutputParser.invoke`（`base.py:204`）把模型输出归一化为文本，调 `parse(text)`。
3. `PydanticOutputParser.parse`（`pydantic.py:84`）先解析 JSON，再 `_parse_obj` 用 pydantic 模型校验，失败抛 `OutputParserException`。

**调用链二：流式增量解析（`BaseCumulativeTransformOutputParser`）**

1. 流式场景下 `_transform` 逐 chunk 累积文本。
2. 每收到新文本调 `parse` 试解析，`_diff(prev, next)` 算出新增部分只增量下发，避免重复输出已解析内容。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `pydantic_object` | `PydanticOutputParser` 的目标模型类 | `pydantic.py` |
| `partial` | `parse_result(..., partial=True)` 允许部分解析（流式） | `base.py:38` |
| `get_format_instructions` | 各解析器自报的 prompt 插入文本 | `base.py:334` |

## 5. 错误与重试语义

- **解析失败**：`PydanticOutputParser` 校验失败经 `_parser_exception`（`pydantic.py:39`）包装成 `OutputParserException`，含原始错误与修正提示。
- **部分解析**：流式下 JSON 未闭合时 `partial=True` 允许返回不完整结构；`partial=False` 抛错。
- **无网络/无重试**：解析是纯本地函数；重试由外层 `with_retry` 或 `with_fallbacks`（见 `core-runnables`）负责。

## 6. 并发细节

- **Runnable 化**：解析器是 `Runnable`，并发/batch 由 `core-runnables` 基类提供。
- **流式无共享锁**：`_transform` 是生成器，累积状态在调用栈局部，背压自然。
- **无异步原语**：默认同步；async 经 Runnable 桥接。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `output_parsers/` 全部：基类与各格式解析器。

**Out-of-Scope（不在本仓库源码内）**
- 模型生成（`core-language-models`/partners）；本叶子只解析其输出。
- pydantic 是第三方依赖（安装，不在本仓库源码内）。

## 8. 与相邻子系统交互

- **上游（`core-language-models`）**：接收模型的 `AIMessage`/`Generation`。
- **本叶子 → 提示词（`core-prompts`）**：`get_format_instructions()` 文本常被拼进 prompt 引导模型按格式输出；`BasePromptTemplate.output_parser` 可挂本叶子实例。
- **本叶子 → 工具（`core-tools`）**：OpenAI function/tools 解析器与工具调用格式对应。
- **本叶子 → Runnable（`core-runnables`）**：解析器本身是 Runnable，可 `|` 进链。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseOutputParser`/`BaseLLMOutputParser` 是抽象基类契约；`get_format_instructions` 是解析器与 prompt 之间的自描述钩子。
- **双轨（完整/流式）**：`BaseCumulativeTransformOutputParser` 提供流式增量解析，与完整 `parse` 双轨。
- **配置驱动**：`pydantic_object`/schema 决定解析目标结构。
- **图类型侧重**：architecture 表达解析器层级与「模型→解析→结构化对象」管道。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 输出解析器层级与解析管道架构图 | `core-output-parsers-architecture.html` | architecture | standard（showcase 对多向继承扇出布局约束严格，降 standard） |

JSON IR 源文件位于 `json/core-output-parsers-architecture.json`。本叶子不补 sequence/dataflow 图。
