# 文档与加载器（core-documents-loaders）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「文档/Blob 数据结构、文档加载器、文档转换器与聊天历史」，不重复展开向量库（见 `../core-vectorstores-retrievers/core-vectorstores-retrievers.md`）与消息（见 `../core-messages/core-messages.md`）。
>
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/documents/` + `document_loaders/` + `chat_loaders.py` + `chat_sessions.py` + `chat_history.py`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `Document` | 文档数据结构：`page_content` + `metadata` | `documents/base.py:288` |
| `Blob` | 二进制数据包装：`as_string`/`as_bytes`/`from_path`/`from_data` | `documents/base.py:59` |
| `BaseMedia` | Document/Blob 共同基类 | `documents/base.py:34` |
| `BaseDocumentTransformer` | 文档转换器 ABC：`transform_documents`（文本切分器基类） | `documents/transformers.py:16` |
| 文档压缩 | `BaseDocumentCompressor`：检索后压缩文档 | `documents/compressor.py` |
| `BaseLoader` | 文档加载器 ABC：`load`/`lazy_load`/`load_and_split` | `document_loaders/base.py:26` |
| `BaseBlobParser` | Blob 解析器 ABC：`lazy_parse`/`parse` | `document_loaders/base.py:117` |
| `BlobLoader` | 二进制 Blob 加载器 ABC：`yield_blobs` | `document_loaders/blob_loaders.py:19` |
| LangSmith 加载 | `langsmith.py`：从 LangSmith 数据集加载 | `document_loaders/langsmith.py` |
| `BaseChatMessageHistory` | 聊天历史 ABC：`add_message`/`add_user_message`/`add_ai_message`/`clear` | `chat_history.py:22` |
| 内存历史 | `InMemoryChatMessageHistory` | `chat_history.py:202` |
| `BaseChatLoader` | 聊天加载器 ABC：`lazy_load -> ChatSession` | `chat_loaders.py:18` |
| `ChatSession` | 一次会话的 TypedDict：messages + functions | `chat_sessions.py:11` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `Document` | `documents/base.py:288` | RAG 检索的基本数据单元；`page_content`(str) + `metadata`(dict) |
| `Blob` | `documents/base.py:59` | 原始字节/字符串包装，`from_path` 读文件，`as_bytes` 取内容 |
| `BaseDocumentTransformer` | `documents/transformers.py:16` | 文本切分/清洗契约；`transform_documents(list[Document])` |
| `BaseLoader` | `document_loaders/base.py:26` | 数据源→`list[Document]` 契约；`lazy_load` 惰性迭代 |
| `BaseBlobParser` | `document_loaders/base.py:117` | `Blob`→`list[Document]` 契约 |
| `BaseChatMessageHistory` | `chat_history.py:22` | 会话历史读写契约；被 `RunnableWithMessageHistory` 使用 |
| `ChatSession` | `chat_sessions.py:11` | 会话数据 TypedDict |

## 3. 关键调用链

**调用链一：加载→切分→入库（RAG 数据准备）**

1. `BaseLoader.load()` 或 `lazy_load()` 从数据源（PDF/网页/数据库）产出 `list[Document]`。
2. `load_and_split`（`base.py:53`）加载后直接经 `BaseDocumentTransformer.transform_documents` 切分。
3. 切分后的小文档送入 `VectorStore.add_documents`（见 core-vectorstores-retrievers）。

**调用链二：Blob 解析（`BaseBlobParser`）**

1. `BlobLoader.yield_blobs` 产出原始 `Blob`。
2. `BaseBlobParser.parse(blob)` 调 `lazy_parse` 流式产出 `list[Document]`。

**调用链三：聊天历史读写（`BaseChatMessageHistory`）**

1. 用户对话时，`add_user_message`/`add_ai_message` 追加消息。
2. `RunnableWithMessageHistory`（见 core-runnables）按 session_id 取 `BaseChatMessageHistory` 实例，invoke 前读历史、invoke 后写新消息。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `Document.metadata` | 来源/页码等元数据 dict | `documents/base.py` |
| `Blob.mimetype`/`path` | Blob 元信息 | `documents/base.py` |
| `load_and_split(text_splitter)` | 加载时用哪个切分器 | `document_loaders/base.py:53` |
| 历史实现 | 内存/Redis/SQL 等由具体子类决定 | `chat_history.py` |

## 5. 错误与重试语义

- **加载失败**：`lazy_load` 是生成器，读取失败在迭代时抛出；`load_and_split` 传播。
- **Blob 校验**：`check_blob_is_valid`（`documents/base.py:151`）在构造期校验路径/数据存在。
- **历史无网络**：`InMemoryChatMessageHistory` 纯内存；持久化后端由子类负责。
- **无自动重试**：本叶子是数据结构/加载契约层。

## 6. 并发细节

- **惰性迭代**：`lazy_load`/`lazy_parse` 是生成器，大文件流式读取，避免一次性载入内存。
- **无共享锁**：`Document`/`Blob` 是值对象；`InMemoryChatMessageHistory` 单线程使用。
- **无异步原语**：本叶子默认同步；异步加载由子类提供。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `documents/` 数据结构与转换器、`document_loaders/` 加载器契约、聊天历史与会话。

**Out-of-Scope（不在本仓库源码内）**
- 具体文件格式解析（PDF/HTML/PPTX 等）在 `langchain` 包或 partners；本叶子只定义 `BaseLoader`/`BaseBlobParser` 契约。
- 持久化历史后端（Redis/SQL 等）在 partners。
- LangSmith 云数据集不在本仓库源码内。

## 8. 与相邻子系统交互

- **上游（数据准备/用户）**：文档加载→切分→入库是 RAG 离线管道。
- **本叶子 → 向量库（`core-vectorstores-retrievers`）**：产出 `list[Document]` 供入库。
- **本叶子 → 消息（`core-messages`）**：`BaseChatMessageHistory` 存取 `list[BaseMessage]`；`ChatSession.messages`。
- **本叶子 → Runnable（`core-runnables`）**：`RunnableWithMessageHistory` 依赖 `BaseChatMessageHistory`。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`BaseLoader`/`BaseDocumentTransformer`/`BaseBlobParser`/`BaseChatMessageHistory` 是抽象基类契约，定义在消费方；实现方在 partners。
- **惰性迭代**：`lazy_load`/`lazy_parse` 是生成器能力缝，大文件流式处理。
- **双轨（eager/lazy）**：`load()`  eagerly 收集 vs `lazy_load()` 惰性迭代双轨。
- **图类型侧重**：architecture 表达「Blob→Loader→Document→Transformer→VectorStore」数据管道。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 文档加载与转换管道架构图 | `core-documents-loaders-architecture.html` | architecture | standard（showcase 对跨行回连边布局约束严格，降 standard） |

JSON IR 源文件位于 `json/core-documents-loaders-architecture.json`。本叶子不补 sequence/dataflow 图。
