# 文档分块工具集（text-splitters）

> 本文是 `text-splitters` 域下的叶子子系统文档。本独立包把长文本切分为适合 LLM 上下文的块。
>
> 源码基准：`langchain_v1` master，commit `89252a8f7043a74f2300729fd038df8221a87e1e`，独立包目录 `libs/text-splitters/langchain_text_splitters/`（13 个 py 文件，共 3686 行）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `TextSplitter` 抽象基类 | 定义分块契约：`chunk_size=4000 / chunk_overlap=200 / length_function=len` | `text_splitter/base.py:59` |
| `split_text` 抽象方法 | 子类实现的文本切分入口 | `text_splitter/base.py:108` |
| `create_documents / split_documents` | 文本/文档列表 → 分块 Document 列表 | `text_splitter/base.py:118,146` |
| `_merge_splits` | 把小片合并为不超过 chunk_size 的块，维护 chunk_overlap 重叠 | `text_splitter/base.py:167` |
| `CharacterTextSplitter` | 单分隔符字符分块 | `character.py:13` |
| `RecursiveCharacterTextSplitter` | 递归多级分隔符（默认 `["\n\n","\n"," ",""]`），优先按语义块切分 | `character.py:91` |
| `from_language` | 按编程语言/标记语言选取专用分隔符集 | `character.py:165` |
| `Language` 枚举 | 各语言（python/jsx/markdown/latex/html...）分隔符表 | `base.py:448` |
| `TokenTextSplitter` | 按 token 计数的分块（tiktoken） | `base.py:325` |
| `from_tiktoken_encoder / from_huggingface_tokenizer` | 用外部 tokenizer 计长的工厂类方法 | `base.py:212,282` |
| 标记语言分块 | Markdown/HTML/LaTeX/JSX/Python/JSON 专用分块器 | `markdown.py / html.py / latex.py / jsx.py / python.py / json.py` |
| 可选分词器集成 | spaCy / NLTK / KoNLPy / sentence-transformers（可选依赖） | `spacy.py / nltk.py / konlpy.py / sentence_transformers.py` |
| 惰性导出 | `__getattr__` 惰性加载符号，避免导入期拉全部重依赖 | `__init__.py:91` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|------|------|------|
| `TextSplitter(BaseDocumentTransformer, ABC)` | `base.py:59` | 分块器基类 |
| `RecursiveCharacterTextSplitter` | `character.py:91` | 最常用的递归分块器 |
| `CharacterTextSplitter` | `character.py:13` | 简单单分隔符分块器 |
| `TokenTextSplitter` | `base.py:325` | token 级分块 |
| `Language` | `base.py:448` | 语言枚举与分隔符映射 |
| `_merge_splits(splits, separator)` | `base.py:167` | 分块合并与重叠核心算法 |

## 3. 关键调用链

**调用链：`RecursiveCharacterTextSplitter.split_text(text)`**

1. 进入 `_split_text(text, ["\n\n","\n"," ",""])`（`character.py:110`）。
2. 遍历分隔符表，选第一个在文本中出现的分隔符 `separator`，并把更细的分隔符放入 `new_separators`（`character.py:116`）。
3. 用正则按 `separator` 切出 `splits`，保留分隔符（`character.py:127`）。
4. 逐片判断：长度 < chunk_size → 进 `good_splits`；否则 → 先把 `good_splits` 经 `_merge_splits` 合并落盘，再对该片用更细的 `new_separators` 递归 `_split_text`（`character.py:134`）。
5. 递归到底仍超长（无新分隔符）则整块保留并告警（`character.py:142`）。
6. `_merge_splits` 累积小片直到超过 chunk_size，再按 chunk_overlap 弹出前缀，产出最终块（`base.py:167`）。

## 4. 配置项

| 配置 | 默认 / 行为 | 位置 |
|------|------|------|
| `chunk_size` | 4000，最大块长 | `base.py:64` |
| `chunk_overlap` | 200，相邻块重叠 | `base.py:65` |
| `length_function` | `len`，可用 tokenizer 计长替换 | `base.py:66` |
| `keep_separator` | 递归分块默认 True | `character.py:101` |
| `separators` | `["\n\n","\n"," ",""]` | `character.py:107` |
| 校验 | chunk_size>0、overlap≥0、overlap≤chunk_size，否则 ValueError | `base.py:88` |

## 5. 错误与重试语义

- **参数非法**：构造期校验 chunk_size/overlap 抛 `ValueError`。
- **超长块告警**：单块超过 chunk_size 时 `logger.warning`，不报错（`base.py:182`）。
- 无网络/重试：本包纯本地文本处理。

## 6. 并发细节

- 纯函数式无状态分块，无锁、无线程。
- `create_documents` 用 `deepcopy` 复制 metadata，避免块间共享可变状态。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 全部分块器实现、合并算法、语言分隔符表、可选分词器集成适配。

**Out-of-Scope（不在本仓库源码内）**
- tiktoken / HuggingFace tokenizer / spaCy / NLTK / KoNLPy / sentence-transformers（外部库）。
- `Document` 类型本体（langchain-core）。

## 8. 与相邻子系统交互

- 上游 RAG 流程 → 本包：加载文档后调用 `splitter.split_documents(...)`。
- 本包 → 下游向量库：分块后的 Document 入嵌入/索引。

## 9. 语言专项适配口径（纯 Python / 能力缝视角）

- **能力缝**：`TextSplitter` 抽象基类定义契约，按"分块策略"能力缝派生出字符/递归/token/各标记语言实现。
- **配置驱动**：chunk_size/overlap/分隔符表/length_function 决定分块行为。
- **可选依赖懒加载**：`__getattr__` 惰性加载，spaCy/NLTK 等重依赖不强制导入。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|------|
| 分块器体系架构图 | `text-splitters-architecture.html` | architecture | showcase |
| 递归分块数据流 | `text-splitters-dataflow.html` | dataflow | showcase |

JSON IR 源文件位于 `json/`。本轮两图均由 standard 提升至 showcase：架构图为"抽象基类→四类实现→可选分词器"主路径，连线清晰；数据流图为"原始文本→选分隔符切分→超长递归→合并输出"四阶段管道，合并→输出流已用 `labelDy` 垂直分离标签。
