# partner-nomic 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文只展开 Nomic 嵌入模型集成（`langchain-nomic`）的职责边界，不重复展开其他嵌入/Chat 模型提供商。
>
> 源码基准：`libs/partners/nomic/`，`langchain-nomic` 包；branch `master`，commit `89252a8f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 文本嵌入 | `NomicEmbeddings.embed_documents` 批量文本向量化 | `langchain_nomic/embeddings.py:116` |
| 查询嵌入 | `embed_query` 单条查询向量化 | `embeddings.py:128` |
| 图像嵌入 | `embed_image` 图像 URI 多模态嵌入 | `embeddings.py:140` |
| 内部统一嵌入 | `embed(texts, task_type)` 按 task_type 分派 | `embeddings.py:97` |

对外导出面（`langchain_nomic/__init__.py`）：`NomicEmbeddings`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `NomicEmbeddings(Embeddings)` | `embeddings.py:13` | 实现 langchain_core `Embeddings` 抽象，封装 Nomic 嵌入 API |
| `embed` | `embeddings.py:97` | 内部统一方法：调 `nomic` SDK，按 `task_type` 参数分派 |

**依赖倒置**：`Embeddings` 抽象在 `langchain_core`，`NomicEmbeddings` 是其实现。

## 3. 关键调用链

### 链 1：文档嵌入

1. `embed_documents(texts)`（:116）调 `self.embed(texts, task_type="search_document")`（:97）。
2. `embed` 调 `nomic` Python SDK 的嵌入方法，SDK 发 HTTPS 到 Nomic API 服务端。
3. 返回向量列表 `list[list[float]]`。

### 链 2：查询嵌入

1. `embed_query(text)`（:128）调 `self.embed([text], task_type="search_query")`。
2. 与文档嵌入走同一 `embed` 方法，仅 `task_type` 不同（Nomic 区分文档/查询嵌入任务）。

### 链 3：图像嵌入

1. `embed_image(uris)`（:140）把图像 URI 列表传给 Nomic 多模态嵌入端点。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `nomic_api_key` | 从 `NOMIC_API_KEY` 环境变量读取 | embeddings.py |
| `model` | 嵌入模型名 | embeddings.py |

## 5. 错误与重试语义

- **API key 缺失**：SDK 初始化时从环境变量读取；缺失时 Nomic API 返回 401。
- **重试**：本层不实现重试；由 `nomic` SDK 负责。

## 6. 并发细节

- **同步方法**：`embed_documents`/`embed_query` 同步；基类 `Embeddings` 提供异步包装。
- **无共享可变状态**：pydantic 实例字段即配置。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `langchain_nomic/embeddings.py`：`NomicEmbeddings` 实现

**Out-of-Scope（不在本仓库源码内）**

- Nomic API 服务端（嵌入推理）——外部服务
- `nomic` Python SDK——依赖项

## 8. 与相邻子系统交互

- **上游 → 本叶子**：向量库（如 Qdrant/Chroma）或 RAG 链通过 `Embeddings` 抽象调用 `NomicEmbeddings`。
- **本叶子 → 下游**：`NomicEmbeddings.embed` → `nomic` SDK → Nomic API 服务端（外部）。
- **同域交互**：依赖 `langchain_core.embeddings.Embeddings` 抽象；与其他 partner 包无直接代码依赖。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：核心能力缝是 `Embeddings` 抽象的 Nomic 实现——与 OpenAIEmbeddings 等并列。
- **配置驱动**：`task_type` 参数（`search_document`/`search_query`）决定嵌入端点行为，同一 `embed` 方法按任务类型分派。
- **外部边界**：Nomic API 服务端、`nomic` SDK 标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Nomic 集成架构图 | `partner-nomic-architecture.html` | architecture | **showcase** |
| 嵌入数据流图 | `partner-nomic-dataflow.html` | dataflow | **showcase** |

**说明**：本叶子不补 sequence 图——嵌入调用是单向管道（文本→嵌入→向量），无多方往返消息交互，与 dataflow 图信息重复。两图均一次通过 showcase 校验。JSON IR 源文件位于 `json/` 目录。
