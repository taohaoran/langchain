# partner-vector-stores 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文归并分析 Qdrant 与 Chroma 两个向量数据库集成，对比其 `VectorStore` 标准接口的实现差异。
> 不重复展开 Chat 模型、Embeddings 等相邻叶子。
>
> 源码基准：`libs/partners/qdrant/`（`langchain-qdrant`）与 `libs/partners/chroma/`（`langchain-chroma`）。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| **Qdrant** | | |
| QdrantVectorStore | 新版向量存储主类，继承 `VectorStore`，支持 dense/sparse/hybrid 检索 | `langchain_qdrant/qdrant.py:36` |
| Qdrant（遗留） | 旧版 `VectorStore` 实现，功能与新版重叠 | `langchain_qdrant/vectorstores.py:60` |
| RetrievalMode | 检索模式枚举（Dense/Sparse/Hybrid） | `qdrant.py:28` |
| add_texts | 文本→embedding→分批 upsert 到 Qdrant collection | `qdrant.py:495` |
| similarity_search | 文本查询→embedding→Qdrant search→Document 列表 | `qdrant.py:520` |
| similarity_search_with_score | 带相关性分数的检索 | `qdrant.py:551` |
| MMR 检索 | max_marginal_relevance_search 系列 | `qdrant.py:728` |
| from_texts / from_existing_collection | 工厂方法：新建集合 / 连接已有集合 | `qdrant.py:339,434` |
| Sparse embeddings | 稀疏向量支持（SparseEmbeddings/SparseVector） | `sparse_embeddings.py` |
| FastEmbed sparse | FastEmbed 稀疏嵌入 | `fastembed_sparse.py` |
| 集合配置校验 | dense/sparse 集合配置验证 | `qdrant.py:1150-1274` |
| **Chroma** | | |
| Chroma | 向量存储主类，继承 `VectorStore`，包装 chromadb.Collection | `langchain_chroma/vectorstores.py:155` |
| add_texts | 文本→embedding function→chroma add | `vectorstores.py:597` |
| similarity_search | 文本查询→chroma query→Document 列表 | `vectorstores.py:730` |
| similarity_search_with_score | 带分数检索 | `vectorstores.py:817` |
| hybrid_search | Chroma 原生混合检索（Search 对象） | `vectorstores.py:684` |
| 图像检索 | add_images / similarity_search_by_image | `vectorstores.py:509,945` |
| fork / reset / delete_collection | 集合管理操作 | `vectorstores.py:492,1129,1124` |
| from_texts / from_documents | 工厂方法 | `vectorstores.py:1273,1375` |

对外导出面：
- `langchain_qdrant/__init__.py`：`QdrantVectorStore`、`Qdrant`、`RetrievalMode`、`SparseEmbeddings`、`SparseVector`、`FastEmbedSparse`
- `langchain_chroma/__init__.py`：`Chroma`

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `QdrantVectorStore(VectorStore)` | `langchain_qdrant/qdrant.py:36` | 新版 Qdrant 向量存储，持有 `QdrantClient` 与 collection_name |
| `Qdrant(VectorStore)` | `langchain_qdrant/vectorstores.py:60` | 旧版实现（与新版 API 重叠） |
| `RetrievalMode(str, Enum)` | `qdrant.py:28` | `DENSE`/`SPARSE`/`HYBRID` 三模式 |
| `SparseEmbeddings` / `SparseVector` | `sparse_embeddings.py` | 稀疏向量类型 |
| `_generate_batches` | `qdrant.py:1038` | 把文本/元数据/ID 分批为 Qdrant points |
| `_build_payloads` / `_build_vectors` | `qdrant.py:1074,1098` | 构建 Qdrant payload 与向量 |
| `_document_from_point` | `qdrant.py:1023` | Qdrant Point → langchain Document |
| `Chroma(VectorStore)` | `langchain_chroma/vectorstores.py:155` | Chroma 向量存储，持有 chromadb.Collection |
| `__query_collection` | `vectorstores.py:450` | 内部查询：embedding→chroma.query→Document |
| `_select_relevance_score_fn` | `vectorstores.py:904` | 选择距离→相关性分数的转换函数 |

**依赖倒置**：两者均继承 `langchain_core.vectorstores.VectorStore` 抽象基类，实现 `add_texts`/`similarity_search`/`from_texts` 等标准接口。

## 3. 关键调用链

### 链 1：Qdrant 写入（add_texts）

1. 用户调用 `store.add_texts(["hello", "world"])`，`QdrantVectorStore.add_texts`（`qdrant.py:495`）。
2. `_generate_batches(texts, metadatas, ids, batch_size=64)`（`:1038`）分批：
   - `_build_vectors`（`:1098`）调用 `self.embeddings.embed_documents(batch_texts)` 得稠密向量（若 sparse 模式则同时构建稀疏向量）；
   - `_build_payloads`（`:1074`）把文本与元数据组装为 Qdrant payload。
3. 每批 `self.client.upsert(collection_name=..., points=...)` 写入 Qdrant 服务端。
4. 返回所有 added_ids。

### 链 2：Qdrant 检索（similarity_search）

1. `similarity_search(query, k=4)`（`qdrant.py:520`）委托 `similarity_search_with_score`（`:551`）。
2. 内部把 query 文本经 `self.embeddings.embed_query(query)` 转向量。
3. `self.client.query_points(...)` 检索，支持 filter/search_params/score_threshold/hybrid_fusion 等 Qdrant 特有参数。
4. `_document_from_point`（`:1023`）把 Qdrant 返回的 ScoredPoint 转回 langchain Document。
5. 去掉分数返回 `list[Document]`。

### 链 3：Chroma 写入与检索

1. `Chroma.add_texts`（`vectorstores.py:597`）：若未提供 ids 则自动生成 UUID（`:620`）；若配置了 embedding function 则 `embed_documents(texts)`。
2. 元数据为空过滤：空 metadata 的条目单独处理（`:634-646`）。
3. 最终调用 chromadb Collection 的 `add(ids, embeddings, documents, metadatas)`。
4. 检索：`__query_collection`（`:450`）把 query 经 embedding function 转向量，调 chroma `collection.query(query_embeddings=..., n_results=k)`，结果映射为 Document。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| **Qdrant** | | |
| `collection_name` | 集合名 | `qdrant.py:210` |
| `batch_size` | `64`（upsert 批大小） | `qdrant.py:500` |
| `content_payload_key` | 文本在 payload 中的字段名 | `qdrant.py` |
| `embeddings` | `Embeddings` 实例或 None | `qdrant.py:273` |
| `vector_name` | 命名向量（多向量集合） | `qdrant.py` |
| retrieval mode | Dense/Sparse/Hybrid | `qdrant.py:28` |
| **Chroma** | | |
| `collection_name` | `"langchain"` | `vectorstores.py:302` |
| `persist_directory` | 持久化目录 | `vectorstores.py:302` |
| `embedding_function` | Chroma 原生 embedding 适配 | `vectorstores.py` |
| `client_settings` | chromadb.Settings | `vectorstores.py:302` |

## 5. 错误与重试语义

- **Qdrant**：`QdrantVectorStoreError`（`qdrant.py:24`）为领域异常；集合配置校验在 `_validate_collection_*`（`:1150-1274`）中发现 dense/sparse 不匹配时抛错。网络重试由 `qdrant-client` SDK 负责。
- **Chroma**：metadata 为空时过滤而非报错（`vectorstores.py:634`）；chromadb 异常透传。
- 两者均不实现应用层重试队列。

## 6. 并发细节

- **分批写入**：Qdrant `add_texts` 按 `batch_size=64` 串行分批 upsert；Chroma 一次性写入（chromadb 内部处理）。
- **无共享可变状态**：实例字段即配置；Qdrant 的 `client` 是 `QdrantClient` 懒属性（`:263`）。
- **线程安全**：依赖底层 SDK 客户端的线程安全语义，本层不加锁。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `langchain_qdrant/`：Qdrant 向量存储适配、稀疏向量、集合校验
- `langchain_chroma/`：Chroma 向量存储适配、图像检索、混合检索

**Out-of-Scope（不在本仓库源码内）**

- Qdrant 服务端（向量数据库存储/检索引擎）——外部服务
- Chroma 服务端（嵌入式或客户端）——外部库/服务
- `qdrant-client` / `chromadb` Python SDK（依赖项）
- Embedding 模型（由用户注入，通常是 `OpenAIEmbeddings` 等其他 partner）
- 相邻叶子：Chat 模型、Embeddings 适配

## 8. 与相邻子系统交互

- **上游 → 本叶子**：用户应用或检索链（`Retriever`、`RAG` 链）通过 `VectorStore` 标准接口调用 `add_texts`/`similarity_search`。
- **本叶子 → 下游**：
  - Qdrant → `qdrant-client` SDK → Qdrant 服务端（外部）；
  - Chroma → `chromadb` 库 → Chroma 持久化/服务端（外部）；
  - 两者都调用用户注入的 `Embeddings` 实例（通常来自 OpenAI 等其他 partner）。
- **同域交互**：依赖 `langchain_core.vectorstores.VectorStore` 抽象；Embeddings 由其他 partner 提供。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：核心能力缝是 **`VectorStore` 抽象基类的标准接口实现**——`add_texts`/`similarity_search`/`similarity_search_with_score`/`from_texts`/`delete`。两个实现把各自数据库的原生 API（Qdrant `upsert`/`query_points`；Chroma `collection.add`/`query`）适配到统一接口。
- **注册表与工厂**：`from_texts`/`from_existing_collection` 是类方法工厂；无字符串→类注册表。
- **可选依赖**：`qdrant-client`/`chromadb` 是硬依赖；Embeddings 是构造函数注入。
- **配置驱动**：Qdrant 的 `RetrievalMode`（Dense/Sparse/Hybrid）决定向量构建与检索路径；Chroma 的 `hybrid_search` 由 Search 对象驱动。
- **实现差异对比**：
  - Qdrant 支持命名向量、稀疏向量、混合检索融合（FusionQuery）、集合配置校验；
  - Chroma 支持图像检索、自动 UUID 生成、fork/reset 集合管理、原生 hybrid_search；
  - Qdrant 有新旧两套实现（`qdrant.py` vs `vectorstores.py`），Chroma 仅一套。
- **外部边界**：Qdrant/Chroma 服务端、SDK 均标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| 向量存储集成架构图 | `partner-vector-stores-architecture.html` | architecture | standard |
| Qdrant 写入/检索数据流图 | `partner-vector-stores-dataflow.html` | dataflow | standard |

降档说明：架构图跨"VectorStore 抽象 ↔ Qdrant/Chroma 适配层 ↔ SDK ↔ 外部数据库"四层，showcase 布局校验未一次通过，降为 standard。JSON IR 源文件位于 `json/` 目录。
