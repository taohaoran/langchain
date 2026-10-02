# 工具集、评估与兼容层（classic-utils-eval）

> 本文是 `classic` 域下的叶子子系统文档（第二轮改进版）。域级总览见 `../classic.md`，本文只展开
> 经典包的 **输出解析器、提示词兼容、评估框架、第三方工具集与散落根级兼容文件**。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `89252a8f`。
>
> 本轮改进：在基线架构图之外，新增"RetryOutputParser 重试解析"时序图，把"解析失败→错误回喂
> LLM→再解析"的容错调用链显式画出。

## 1. 功能清单

源码位于 `utilities/`（58 文件）、`evaluation/`（32 文件）、`output_parsers/`（23 文件）、
`prompts/`（12 文件）及根级兼容文件。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 输出解析器族 | 把 LLM 文本输出解析成结构化值 | `output_parsers/` |
| 重试解析器 | 解析失败后把错误回喂 LLM 重试 | `output_parsers/retry.py:54` |
| Pydantic/Structured/JSON/Enum/XML/YAML 解析器 | 各类结构化输出解析 | `output_parsers/pydantic.py`、`structured.py`、`json.py` 等 |
| 提示词模板兼容 | 重导出 core 的提示词基类 + few-shot | `prompts/base.py`、`prompts/few_shot.py` |
| 示例选择器 | few-shot 示例动态选择 | `prompts/example_selector/` |
| 评估框架 | LLM 作为评审者的各评测链 | `evaluation/` |
| 标准评估 | 按准则打分 / 带标签准则 | `evaluation/criteria/eval_chain.py:162`、`:508` |
| 其他评测维度 | 问答、对比、精确匹配、正则、embedding 距离、解析 | `evaluation/{qa,comparison,exact_match,regex_match,embedding_distance,parsing}/` |
| 第三方工具集 | 58 个工具集，57 个为 community 转发桩 | `utilities/` |
| SerpAPI/Requests/PythonREPL 兼容 | 根级 shim | `serpapi.py`、`requests.py`、`python.py` |
| 格式化/示例生成/模型实验室 | 根级兼容与工具 | `formatting.py`、`example_generator.py`、`model_laboratory.py` |
| SQL 数据库兼容 | shim 指向 community | `sql_database.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `BaseOutputParser` | 来自 `langchain_core` | 能力缝：`parse(text)` → 结构化值；`get_format_instructions` |
| `RetryWithErrorOutputParser` | `output_parsers/retry.py:187` | 解析失败时把错误拼回提示词让 LLM 重生成 |
| `BasePromptTemplate` | `prompts/base.py`（重导出 core） | 提示词模板契约 |
| `CriteriaEvalChain` | `evaluation/criteria/eval_chain.py:162` | 用 LLM 按准则给输出打分 |
| `StringEvaluator` / `LLMEvalChain` | `evaluation/` | 评估链基类 |
| `create_importer` + `DEPRECATED_LOOKUP` | 各 `utilities/*.py`、根 shim | 延迟导入 community |

## 3. 关键调用链

**输出解析重试（对应新增重试时序图）**：`RetryWithErrorOutputParser.parse`
（`output_parsers/retry.py:187`）→ 首次解析失败捕获 `OutputParserException` →
把原始输出 + 错误信息拼进重试提示词 → 再调 LLM → 再次解析。这是"解析失败即回喂"的容错能力缝。

**LLM 评审**：`CriteriaEvalChain.evaluate_strings(prediction=..., input=...)`（
`evaluation/criteria/eval_chain.py:162`）→ 构造评审提示词 → 调 LLM → 解析出 score/理由。

**工具集访问**：`from langchain_classic.utilities import SerpAPIWrapper` → `__getattr__` →
`create_importer` 从 `langchain_community.utilities` 加载并告警。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| 解析器 `pydantic_schema` / `output_keys` | 由具体解析器定义 | `output_parsers/pydantic.py` 等 |
| 评估准则 | 字符串准则或预置准则名 | `evaluation/criteria/` |
| 工具集 API key | 环境变量（如 SERPAPI_API_KEY） | 各 utilities（community 内） |
| `DEPRECATED_LOOKUP` | 每文件字符串→community 映射 | 各 `utilities/*.py` |

## 5. 错误与重试语义

- **解析失败**：普通解析器直接抛 `OutputParserException`；`RetryWithErrorOutputParser`
  把错误回喂 LLM 重试一次。
- **缺失可选依赖**：工具集/第三方库延迟导入抛错并给安装指引。
- **评估**：评审 LLM 输出不合规时按解析器策略处理。
- 本层不做网络重试；第三方 API 调用重试由 community 实现负责。

## 6. 并发细节

本叶子多为纯函数式解析与同步工具封装，**无自建并发模型**。评估链串行评审；工具集同步
调用外部 API。异步由 core Runnable 派生包装。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 输出解析器实现、few-shot 提示词、评估链、工具集导入桩、根级兼容文件。

**Out-of-Scope（不在本仓库源码内）**
- `BaseOutputParser`/`BasePromptTemplate` 在 `langchain-core`。
- 57/58 工具集真实实现在 **`langchain-community`**。
- 第三方 API（SerpAPI、Google、GitHub 等）为外部服务。
- Pydantic、jinja2 等为第三方库。

## 8. 与相邻子系统交互

- 上游：`LLMChain`（classic-chains）调用输出解析器；用户调用评估链与工具集。
- 本叶子 → 下游：
  - 解析器 → `langchain_core` 输出类型。
  - 评估链 → LLM（classic-llms-chat/partner）。
  - 工具集 → `langchain-community` → 外部 API。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseOutputParser` 是输出解析能力缝；评估链是"LLM 自评审"能力缝；
  few-shot 示例选择器是"示例动态供给"能力缝。
- **目录即实例库**：utilities 58 文件几乎全是同构转发桩（57/58），深读 1 个代表即可推断。
- **可选依赖与懒加载**：`create_importer` 延迟加载第三方工具。
- **与 core/v1 的关系**：输出解析器与提示词基类已在 core；工具集已迁 community；本包为
  兼容与少量内置实现。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 解析/评估/工具集体系架构图 | `classic-utils-eval-architecture.html` | architecture | showcase |
| RetryOutputParser 重试解析时序图 | `classic-utils-eval-retry-sequence.html` | sequence | showcase |

档位披露：架构图组件职责清晰，保持 `showcase`。新增时序图刻画"Chain→LLM→重试解析器→
内层解析器"的解析—失败—回喂—再解析交互；本轮通过移除显式 viewBox 让渲染器自动适配画布，由 `standard` 提升至 `showcase`。JSON IR 位于 `json/`。
