# 工具抽象（core-tools）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「供模型调用的工具/函数抽象」，不重复展开 Runnable（见 `../core-runnables/core-runnables.md`）与消息（见 `../core-messages/core-messages.md`）。
>
> 源码基准：`langchain-core` master，commit `89252a8f7043a74f2300729fd038df8221a87e1e`，源码位于 `libs/core/langchain_core/tools/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `BaseTool` | 工具基类，继承 `RunnableSerializable[str|dict|ToolCall, Any]`，工具即 Runnable | `tools/base.py:433` |
| 工具调用入口 | `invoke`/`run`/`_run`：解析输入 schema 后执行函数体 | `base.py:757`、`1009`、`909` |
| schema 推断 | `create_schema_from_function`：从函数签名 + docstring 推断参数 pydantic schema | `base.py:263` |
| docstring 解析 | `_parse_python_function_docstring`/`_infer_arg_descriptions` 提取参数描述 | `base.py:126`、`170` |
| 工具异常 | `ToolException`：工具执行失败，可经 `handle_tool_error` 控制反馈 | `base.py:371` |
| `StructuredTool` | 接受 dict/多参数输入的结构化工具 | `tools/structured.py:93` |
| `Tool` | 单字符串输入的简单工具 | `tools/simple.py:31` |
| `@tool` 装饰器 | 把函数/Runnable 装饰成 `BaseTool` | `tools/convert.py:18` |
| Runnable 转工具 | `convert_runnable_to_tool` | `tools/convert.py:433` |
| 检索器转工具 | `create_retriever_tool`：把 retriever 包成可调用工具 | `tools/retriever.py:31` |
| 工具定义渲染 | `render.py`：把工具列表渲染成模型所需格式（OpenAI tools 等） | `tools/render.py` |
| 注入参数 | `_injected_args_keys`/`_filter_injected_args`：把 run_manager/config 等注入函数而不暴露给模型 | `base.py:725`、`934` |
| 输入解析 | `_parse_input`/`_to_args_and_kwargs`：把 ToolCall/dict 归一化为位置/关键字参数 | `base.py:778`、`970` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `BaseTool` | `tools/base.py:433` | 工具契约；持有 `args_schema`、`name`、`description`；`__init_subclass__` 钩子做子类注册 |
| `ToolException` | `tools/base.py:371` | 工具执行期异常 |
| `create_schema_from_function` | `tools/base.py:263` | 从 Python 函数签名自动生成 pydantic 参数模型 |
| `StructuredTool` | `tools/structured.py:93` | 通用结构化工具；`from_function` 工厂 |
| `Tool` | `tools/simple.py:31` | 单输入字符串工具 |
| `@tool` | `tools/convert.py:18` | 装饰器，把函数包装成工具，自动推断 schema |
| `create_retriever_tool` | `tools/retriever.py:31` | 检索器→工具工厂 |
| `convert_runnable_to_tool` | `tools/convert.py:433` | Runnable→工具 |

## 3. 关键调用链

**调用链一：模型发起工具调用到执行（`BaseTool.invoke`，`base.py:757`）**

1. 模型产出 `AIMessage.tool_calls`（含 name + args）。
2. 经 `BaseTool.invoke`，`_parse_input`（`base.py:778`）把 ToolCall/dict 按 `args_schema` 校验解析。
3. `_to_args_and_kwargs`（`base.py:970`）拆成位置/关键字参数；`_filter_injected_args`（`base.py:934`）剔除注入参数（run_manager 等）。
4. 调 `_run`/`_arun` 执行函数体；`_handle_tool_error`（`base.py:1313`）把异常转成可读错误反馈。
5. `_format_output`（`base.py:1390`）把结果格式化成字符串/内容块，包成 `ToolMessage` 回传模型。

**调用链二：`@tool` 装饰器把函数变工具（`convert.py`）**

1. `@tool` 装饰函数，`_create_tool_factory`（`convert.py:270`）调用 `create_schema_from_function` 推断参数 schema。
2. 生成 `StructuredTool`，name 取函数名、description 取 docstring。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `args_schema` | 工具参数 pydantic 模型；缺省由 `create_schema_from_function` 推断 | `base.py` |
| `return_direct` | 工具执行后是否直接返回给用户、不再回模型 | `base.py` |
| `handle_tool_error` | 异常处理策略（bool/str/callable） | `base.py:1313` |
| `injected_args` | 不暴露给模型、由运行时注入的参数键 | `base.py:725` |

## 5. 错误与重试语义

- **schema 校验失败**：模型传参不符 schema，`_handle_validation_error`（`base.py:1281`）把校验错误转回字符串反馈给模型，让模型自我纠正而非中断。
- **工具执行异常**：`ToolException` 或普通异常经 `handle_tool_error` 处理；配置为字符串时返回该固定提示，callable 时由调用方决定。
- **无自动重试**：工具本身不重试；外层可包 `with_retry`（见 `core-runnables`）。

## 6. 并发细节

- **Runnable 化**：工具是 `RunnableSerializable`，并发/batch/stream 由 `core-runnables` 提供；`_arun` 异步路径原生并发。
- **无共享锁**：单例工具若函数本身线程安全则并发安全；注入参数经 contextvars 传递。
- **__init_subclass__**：子类定义时即做 schema 校验，把错误提前到导入期。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `tools/` 全部：工具基类、schema 推断、装饰器、检索器转工具、渲染。

**Out-of-Scope（不在本仓库源码内）**
- 模型如何决定调哪个工具（agent loop）在 `langchain_v1`/agent 层；本叶子只提供可调用工具抽象。
- 实际工具执行的外部 API（搜索、数据库等）在工具实现方。

## 8. 与相邻子系统交互

- **上游（模型/agent）**：模型依据 `args_schema`/`description` 发起 `tool_calls`；本叶子执行后回 `ToolMessage`。
- **本叶子 → Runnable（`core-runnables`）**：`BaseTool` 继承 `RunnableSerializable`；`Runnable.as_tool()` 把任意 Runnable 包成工具。
- **本叶子 → 消息（`core-messages`）**：执行结果格式化为 `ToolMessage`；输入解析 `ToolCall`。
- **本叶子 → 检索器（`core-vectorstores-retrievers`）**：`create_retriever_tool` 包装检索器。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseTool` 是抽象基类契约；`@tool`/`from_function`/`create_retriever_tool` 是工厂/装饰器扩展点；`handle_tool_error` 是可插拔错误钩子。
- **注册表**：工具的 `name` + `args_schema` 构成模型可发现的能力表，agent 按 name 路由到工具。
- **配置驱动**：`args_schema` 决定模型可见参数；`injected_args` 隐藏运行时参数。
- **图类型侧重**：architecture 表达工具抽象与类层级，sequence 表达「模型发起 tool_call → 解析校验 → 执行函数体 → ToolMessage 回传」的多方消息交互时序；无循环状态机，不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 工具抽象与调用管道架构图 | `core-tools-architecture.html` | architecture | **showcase**（本轮移除多条重复的「继承/校验输入」边标签，解决 `label_route_clearance`，由 standard 提升至 showcase） |
| 工具调用执行时序图 | `core-tools-sequence.html` | sequence | **showcase**（本轮新增；模型/BaseTool/args_schema/_run 四方消息交互，符合 sequence 核心语义） |

JSON IR 源文件位于 `json/` 目录。本轮相对基线的改进：架构图移除重复边标签后由 standard 提升至 showcase；新增 1 张 sequence 图表达工具调用执行时序。两图均实际渲染成功（退出码 0、HTML 非空）。
