# 向量库与检索器（core-vectorstores-retrievers）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「向量存储、嵌入模型、检索器与增量索引」的抽象基类，不重复展开文档（见 `../core-documents-loaders/core-documents-loaders.md`）。
>
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/vectorstores/` + `retrievers.py` + `embeddings/` + `indexing/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| `VectorStore` | 向量存储 ABC：增删查文档 | `vectorstores/base.py:43` |
| 写入 | `add_texts`/`add_documents`/`from_texts`/`from_documents` | `base.py:46`/`234`/`848`/`787` |
| 相似度检索 | `similarity_search`/`similarity_search_with_score`/`similarity_search_by_vector` | `base.py:361`/`417`/`624` |
| MMR 检索 | `max_marginal_relevance_search`/`..._by_vector`：兼顾相关性与多样性 | `base.py:659`/`724` |
| 相关性分数 | `_select_relevance_score_fn`：把距离归一化成 [0,1] 相关分 | `base.py:403` |
| 转检索器 | `as_retriever`：把向量库包成 `VectorStoreRetriever` | `base.py:905` |
| `VectorStoreRetriever` | 向量库检索器实现（`BaseRetriever` 子类） | `base.py:964` |
| `BaseRetriever` | 检索器 ABC，`RunnableSerializable[RetrieverInput, RetrieverOutput]` | `retrievers.py:55` |
| 检索契约 | 抽象 `_get_relevant_documents`；`invoke` 提供 Runnable 包装 | `retrievers.py:298`、`179` |
| `Embeddings` | 嵌入模型 ABC：`embed_documents`/`embed_query` | `embeddings/embeddings.py:8` |
| 增量索引 | `index`：基于内容 hash 的文档增删改同步，避免重复写入 | `indexing/api.py:296` |
| 索引结果 | `IndexingResult`（已加/更新/跳过/删除数）；`_HashedDocument` | `api.py:283`/`235` |
| 哈希 | `_hash_string`/`_hash_nested_dict`：文档内容指纹 | `api.py:73`/`83` |
| 内存向量库 | `InMemoryVectorStore`（测试/本地用） | `vectorstores/in_memory.py` |
| 内存嵌入 | `FakeEmbeddings` | `embeddings/fake.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `VectorStore` | `vectorstores/base.py:43` | 向量库契约；`add_texts` 与 `_similarity_search` 为核心抽象 |
| `BaseRetriever` | `retrievers.py:55` | 检索器契约；本身是 Runnable，输入 query 输出 `list[Document]` |
| `VectorStoreRetriever` | `vectorstores/base.py:964` | 包装 VectorStore 的检索器，`search_type`(similarity/mmr) 可配 |
| `Embeddings` | `embeddings/embeddings.py:8` | 嵌入模型契约；`embed_documents`/`embed_query` |
| `index` | `indexing/api.py:296` | 增量索引主函数：对比新旧文档 hash 集合作增删改 |
| `_HashedDocument` | `indexing/api.py:235` | 带内容 hash 的文档包装 |

## 3. 关键调用链

**调用链一：检索（`retriever.invoke(query)`，`retrievers.py:179`）**

1. 经 Runnable 执行链（见 core-runnables），`_get_relevant_documents(query)` 被子类实现。
2. `VectorStoreRetriever` 调 `VectorStore.similarity_search`/`max_marginal_relevance_search`，先 `Embeddings.embed_query(query)` 把查询向量化，再在向量库 ANN 检索。
3. 返回 `list[Document]`，经 `similarity_search_with_relevance_scores` 可附相关分。

**调用链二：写入（`vector_store.add_documents`，`base.py:234`）**

1. 接收 `list[Document]`，内部 `Embeddings.embed_documents([d.page_content for d in docs])` 批量向量化。
2. `add_texts` 把向量 + 元数据写入向量库，返回 id 列表。

**调用链三：增量索引（`index`，`indexing/api.py:296`）**

1. 对新文档集算 `_HashedDocument` 内容 hash。
2. 与向量库中已存 hash 集合对比：新增的加、变更的更新、消失的删除、未变的跳过。
3. 返回 `IndexingResult`（记录各类计数），避免重复嵌入与写入。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `search_type` | `similarity`/`mmr`/`similarity_score_threshold` | `base.py:988` |
| `k` | 返回文档数 | `similarity_search` |
| `fetch_k`/`lambda_mult` | MMR 的候选数与多样性权重 | `base.py:659` |
| `score_threshold` | 相似度阈值过滤 | `base.py:506` |
| `index(mode)` | `incremental`/`full`/`update`/`upsert`/`delete` | `api.py:296` |
| `source_id_key` | 用文档哪个字段做来源 id | `api.py:122` |

## 5. 错误与重试语义

- **检索失败**：向量库查询失败经回调 `on_retriever_error` 上报；异常上抛。
- **嵌入失败**：`embed_documents` 失败中断写入。
- **哈希一致性**：`_warn_about_sha1` 提示弱哈希；增量索引依赖内容 hash 稳定。
- **无自动重试**：本叶子是抽象层；重试由外层 `with_retry` 负责。

## 6. 并发细节

- **Runnable 化**：`BaseRetriever` 是 `RunnableSerializable`，并发/batch 由 `core-runnables` 提供。
- **批量嵌入**：`embed_documents` 接收列表，底层可批量调 API；`embed_query` 单查询。
- **无共享锁**：检索是只读；写入时向量库实现方负责并发（本叶子不约束）。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `vectorstores/` 抽象基类与内存实现、`retrievers.py`、`embeddings/` 契约与假实现、`indexing/` 增量索引。

**Out-of-Scope（不在本仓库源码内）**
- 实际向量数据库（Chroma/Pinecone/FAISS/PGVector 等）在 partners；本叶子只定义 `VectorStore` 契约。
- 嵌入模型 API（OpenAI embeddings 等）在 partners。
- ANN 索引算法实现在各向量库后端。

## 8. 与相邻子系统交互

- **上游（RAG 链）**：`retriever | prompt | model`；检索结果 `list[Document]` 注入 prompt。
- **本叶子 → 文档（`core-documents-loaders`）**：操作 `Document` 对象（page_content/metadata）。
- **本叶子 → 模型（`core-language-models`）**：`Embeddings` 是嵌入模型契约，与 chat model 同级。
- **本叶子 → 工具（`core-tools`）**：`create_retriever_tool` 把检索器包成工具。
- **本叶子 → 回调（`core-callbacks-tracers`）**：`on_retriever_start/end/error` 包裹检索。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：`VectorStore`/`BaseRetriever`/`Embeddings` 是抽象基类契约，定义在消费方；实现方在 partners（依赖倒置）。
- **Runnable 化**：`BaseRetriever` 直接是 Runnable，检索即组合。
- **注册表/工厂**：`as_retriever` 是 VectorStore→Retriever 工厂；`search_type` 字符串选择检索策略。
- **配置驱动**：`search_type`/`k`/`score_threshold`/`mode` 驱动检索与索引行为。
- **图类型侧重**：architecture 表达向量库/嵌入/检索器三角与「query→embed→ANN→Document」管道。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 向量库/嵌入/检索器架构与检索管道图 | `core-vectorstores-retrievers-architecture.html` | architecture | showcase |

JSON IR 源文件位于 `json/core-vectorstores-retrievers-architecture.json`。本叶子不补 sequence/dataflow 图。
