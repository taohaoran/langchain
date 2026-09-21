# 文档加载与文本分割（classic-loaders）

> 本文是 `classic` 域下的叶子子系统文档。域级总览见 `../classic.md`，本文只展开经典包的
> **文档加载器集合、文档转换器与文本分割兼容层**，不展开向量索引（见
> `../classic-retrievers-stores/`）。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `4492ad7a8`。

## 1. 功能清单

本叶子提供把外部数据载入为 `Document`、再切分为块的能力。源码位于
`document_loaders/`（166 文件）、`document_transformers/`（11 文件）、`text_splitter.py`。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 加载器基类重导出 | `BaseLoader`/`BaseBlobParser` 仅 3 行 re-export | `document_loaders/base.py` |
| 文本分割器重导出 | 全部 Splitter 从 `langchain_text_splitters` 重导出 | `text_splitter.py:3` |
| 文档加载器集合 | ~147 个加载器，其中 144 个为 `create_importer` 转发桩 | `document_loaders/` |
| 文档转换器 | 11 个转换器，全部转发桩 | `document_transformers/` |

**加载器覆盖（目录即实例库，深读代表 + 共性列表）**：PDF（PDFMiner/PyMuPDF/OnlinePDF…）、
CSV/JSON/Markdown/HTML、Notion、Confluence、Google Drive、S3/Azure Blob、GitHub/GitLab、
YouTube/Bilibili、ArXiv、维基、网页（AsyncHTML/Chromium/Browserless）等，均经
`create_importer` 指向 `langchain_community.document_loaders`。

**分割器**（`text_splitter.py` 重导出自 `langchain_text_splitters`）：
`RecursiveCharacterTextSplitter`、`CharacterTextSplitter`、`TokenTextSplitter`、
`MarkdownHeaderTextSplitter`、`HTMLHeaderTextSplitter`、`PythonCodeTextSplitter`、
`LatexTextSplitter` 等。

**转换器**：BeautifulSoup、Doctran（抽取/问答/翻译）、HTML2Text、Google Translate、
LongContextReorder、冗余过滤等。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `BaseLoader` | `document_loaders/base.py:1`（转发自 core） | 能力缝：`load()` / `lazy_load()` 返回 `list[Document]` |
| `BaseBlobParser` | `document_loaders/base.py:1` | 二进制块解析契约 |
| `TextSplitter` / `RecursiveCharacterTextSplitter` | `text_splitter.py`（转发自 `langchain_text_splitters`） | 把 Document 切成块的契约与递归字符实现 |
| `create_importer` + `DEPRECATED_LOOKUP` | 各 `document_loaders/<x>.py` | 延迟导入桩：访问时从 `langchain_community` 加载 |

## 3. 关键调用链

**文档加载管道（概念流，与 dataflow 图对应）**：

1. 用户选某个加载器类（如 `PDFMinerLoader`）→ 触发模块 `__getattr__` → `create_importer`
   从 `langchain_community.document_loaders` 动态加载真实类并告警。
2. 实例化加载器（传入文件路径/连接参数）→ 调 `loader.load()` → 返回 `list[Document]`。
3. 把 Document 交给 `RecursiveCharacterTextSplitter.split_documents(...)` → 按分隔符递归
   切分、控制 `chunk_size`/`chunk_overlap` → 得到块级 Document。
4. 块 Document 通常送入向量索引（见 `classic-retrievers-stores/`）。

**文本分割核心**（实际在 `langchain_text_splitters`，不在本仓库）：
`RecursiveCharacterTextSplitter` 按 `["\n\n", "\n", " ", ""]` 分隔符递归切分，
超长块再细分，相邻块保留 `chunk_overlap` 重叠。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `chunk_size` / `chunk_overlap` | 切分块大小与重叠（定义在 `langchain_text_splitters`） | 不在本仓库源码内 |
| 各加载器构造参数 | 由 community 实现定义（路径、凭证、模式） | 不在本仓库源码内 |
| `DEPRECATED_LOOKUP` | 每文件内字符串→community 模块映射 | 各 `document_loaders/*.py` |

## 5. 错误与重试语义

- **缺失可选依赖**：访问某加载器时若 `langchain_community` 或其解析库（pypdf、bs4 等）
  未安装，延迟导入抛出带安装指引的错误。
- **加载失败**：本层不做重试，文档解析异常向上抛。
- **弃用告警**：旧路径导入触发 `LangChainDeprecationWarning`。
- 分割器纯内存处理，无外部失败路径。

## 6. 并发细节

本叶子为转发/重导出层，**无自建并发模型**。真实加载器可能用 `lazy_load` 惰性迭代器
逐文件加载、或 `concurrent.py` 并发调度；这些在 `langchain_community` 实现。本层只提供
导入路径。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `document_loaders/base.py`、`document_transformers/`、`text_splitter.py` 的兼容转发；
  147 个加载器导入桩。

**Out-of-Scope（不在本仓库源码内）**
- `BaseLoader` 基类在 `langchain-core`；分割器实现在独立包 `langchain-text-splitters`。
- 全部加载器/转换器真实实现在 **`langchain-community`**。
- 外部数据源（S3、Google Drive、Notion、YouTube、数据库等）本身不在本仓库源码内。
- PDF/HTML 解析库（pypdf、beautifulsoup4 等）为第三方，不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游：用户代码 / 索引构建脚本 → 选择加载器与分割器。
- 本叶子 → 下游：
  - `BaseLoader`（core）→ `langchain_community` 加载器 → 外部数据源。
  - `TextSplitter`（`langchain-text-splitters`）。
  - 输出块 Document → 向量存储/索引（`classic-retrievers-stores/`）做嵌入与检索。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseLoader` 是加载能力缝（`load`/`lazy_load`）；`TextSplitter` 是分割
  能力缝。本叶子只做旧路径转发。
- **目录即实例库**：147 个加载器是典型"目录即实例库"——绝大多数为同构 `create_importer`
  转发桩，深读 `pdf.py` 1 个代表即可推断全部，其余以类别列表说明。
- **可选依赖与懒加载**：`create_importer` 延迟加载 community 与各解析库，顶层 import 轻量。
- **与 core/v1 的关系**：基类已在 core、分割器已独立为 `langchain-text-splitters`、
  加载器已迁至 community，本包仅保留旧导入路径。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 文档加载→分割管道数据流图 | `classic-loaders-dataflow.html` | dataflow | showcase |

降档披露：管道节点少、流向单一，保持 `showcase`。架构图不另出（与叶子 3 同为转发层，
数据图已表达主路径）。JSON IR 位于 `json/`。
