# 检索器、向量存储与索引（classic-retrievers-stores）

> 本文是 `classic` 域下的叶子子系统文档（第二轮改进版）。域级总览见 `../classic.md`，本文只展开
> 经典包的 **检索器组合、向量存储兼容层、文档/键值存储与索引 API**，不展开文档加载（见
> `../classic-loaders/`）与检索问答链（见 `../classic-chains/`）。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `89252a8f`。
>
> 本轮改进：架构图在当前 archify 版本下由基线 `standard` 提升至 `showcase`；新增"检索数据流"
> 图，把查询经组合检索器下推向量库、召回后压缩的管道画出来。

## 1. 功能清单

源码位于 `retrievers/`（78 文件）、`vectorstores/`（75 文件）、`docstore/`（6 文件）、
`storage/`（8 文件）、`indexes/`（9 文件）。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 向量存储基类重导出 | `VectorStore`/`VectorStoreRetriever` 仅 3 行 shim | `vectorstores/base.py` |
| 向量库实现转发 | ~75 个向量库文件，66 个为 `create_importer` 桩 | `vectorstores/` |
| 检索器基类 | `BaseRetriever`（core）；本包提供组合型检索器 | `retrievers/` |
| 上下文压缩检索器 | 对召回结果做压缩/过滤 | `retrievers/contextual_compression.py:13` |
| 集成检索器 | 多路检索器结果融合 | `retrievers/ensemble.py:53` |
| 多查询检索器 | LLM 生成多个查询并行检索 | `retrievers/multi_query.py:49` |
| 多向量/父文档检索 | 子块嵌入、召回后回溯父文档 | `retrievers/multi_vector.py:29`、`parent_document_retriever.py:11` |
| 自查询检索器 | LLM 把自然语言转成元数据过滤 | `retrievers/self_query/` |
| 文档存储 | `Docstore` 转发 community + 内存实现 | `docstore/` |
| 键值存储 | `ByteStore` 多后端：内存/文件/Redis | `storage/in_memory.py`、`file_system.py:12`、`redis.py` |
| 编码器后端存储 | 用 encoder 把 key 哈希后存储 | `storage/encoder_backed.py:13` |
| 索引封装 | `VectorStoreIndexWrapper` 便捷问答 | `indexes/vectorstore.py` |
| 记录管理器 | SQL 记录去重/增量索引 | `indexes/_sql_record_manager.py:85` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `VectorStore` / `VectorStoreRetriever` | `vectorstores/base.py`（转发自 core） | 向量增删/相似度检索能力缝 |
| `BaseRetriever` | core | 检索能力缝：`_get_relevant_documents` |
| `ContextualCompressionRetriever` | `retrievers/contextual_compression.py:13` | 组合检索器 + 压缩器 |
| `EnsembleRetriever` | `retrievers/ensemble.py:53` | 多检索器 RRF 融合 |
| `MultiVectorRetriever` / `ParentDocumentRetriever` | `retrievers/multi_vector.py:29` | 子块召回→父文档回溯 |
| `BaseStore` / `ByteStore` | core + `storage/` | key-value 存取能力缝 |
| `LocalFileStore` | `storage/file_system.py:12` | 本地文件系统 ByteStore |
| `VectorStoreIndexWrapper` | `indexes/vectorstore.py` | 向量库 + 问答链便捷封装 |
| `SQLRecordManager` | `indexes/_sql_record_manager.py:85` | 增量索引去重记录 |

## 3. 关键调用链

**检索（典型，对应新增检索数据流图）**：`RetrievalQA`（见 classic-chains）→
`retriever.get_relevant_documents(q)` → 组合型检索器可能：多查询扩展（MultiQuery）→
各子检索器召回 → 上下文压缩 → 去重合并 → 返回相关 Document。

**向量库写入**：`VectorStore.from_texts(texts, embedding)` → 调 `embedding.embed_documents`
→ `add_texts` 写入外部向量库（community 实现）。

**增量索引**：`indexes/_api.py` 的索引流程 → `SQLRecordManager` 记录已处理 doc hash →
仅对变更文档重嵌入写入，避免重复。

**键值存储**：`ByteStore.mset/mget/mdelete` 抽象，`LocalFileStore` 写本地、`Redis` 写外部 Redis。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `chunk_size`/`overlap`（索引封装默认） | `RecursiveCharacterTextSplitter(1000, 0)` | `indexes/vectorstore.py:_get_default_text_splitter` |
| `search_kwargs`（检索 top_k） | 由调用方传入 | 各检索器 |
| 向量库连接参数 | 由 community 实现定义 | 不在本仓库源码内 |
| record manager 数据库 | SQL 连接串 | `indexes/_sql_record_manager.py` |

## 5. 错误与重试语义

- **缺失可选依赖**：访问某向量库/Redis 时延迟导入抛错并给安装指引。
- **检索失败**：组合检索器中单路子检索器失败的处理取决于具体实现；本层默认向上抛。
- **记录管理**：SQL 操作失败向上抛，不自动重试。
- 本层不做向量检索重试/退避。

## 6. 并发细节

- 组合检索器（`EnsembleRetriever`、`MultiQueryRetriever`）可并发调用子检索器；实际并发
  调度在实现内。
- 键值存储无自建线程；`LocalFileStore` 文件 IO 同步。
- 无显式锁；外部向量库/Redis 的并发语义由后端保证。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 组合型检索器、键值存储后端、索引封装与记录管理器；各基类 shim。

**Out-of-Scope（不在本仓库源码内）**
- `VectorStore`/`BaseRetriever` 基类在 `langchain-core`。
- 全部向量库实现（Chroma/Pinecone/FAISS/Milvus/Weaviate 等）在 **`langchain-community`**。
- 外部向量数据库、Redis、文件系统本身不在本仓库源码内。
- 嵌入模型调用见 `embeddings/`（转发层）与 partner 包。

## 8. 与相邻子系统交互

- 上游：`RetrievalQA`（classic-chains）、`VectorStoreIndexWrapper`、用户代码 → 调检索器。
- 本叶子 → 下游：
  - 向量库/检索器 → `langchain-community` → 外部向量数据库。
  - 嵌入 → `langchain_core.embeddings.Embeddings`（embeddings/ 转发层）。
  - 文档加载 → `classic-loaders/` 的 Document 汇入索引。
  - 键值存储 → 外部 Redis/文件系统。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`VectorStore`、`BaseRetriever`、`ByteStore` 是三条独立能力缝；组合型
  检索器（Ensemble/MultiQuery/MultiVector）是"装饰器/组合模式"能力缝——包装一个检索器
  增强召回，而非新增存储。
- **目录即实例库**：vectorstores 75 文件、retrievers 44 文件中大量是同构转发桩（66/44），
  深读代表 + 共性列表。
- **可选依赖与懒加载**：向量库/Redis 第三方客户端延迟 import。
- **与 core/v1 的关系**：基类已在 core、向量库实现已迁出 community，本包保留组合检索器
  与存储/索引实用类，旧导入路径兼容。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 检索/向量/存储体系架构图 | `classic-retrievers-stores-architecture.html` | architecture | showcase |
| 检索数据流图 | `classic-retrievers-stores-retrieval-dataflow.html` | dataflow | showcase |

档位披露：架构图在当前 archify 版本下通过 showcase（相对基线 `standard` 为提升项）。
新增数据流图刻画"查询 → 组合检索器 → 向量库 shim → community/外部向量库 → 召回文档 →
压缩"管道；本轮通过移除显式 viewBox 让渲染器自动适配，由 `standard` 提升至 `showcase`。JSON IR 位于 `json/`。
