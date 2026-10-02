# partner-qdrant 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文只展开 Qdrant 向量数据库集成（`langchain-qdrant`）的职责边界，不重复展开 Chroma 等其他向量库、
> Chat 模型提供商、Exa 搜索等相邻叶子。
>
> 源码基准：`libs/partners/qdrant/`，`langchain-qdrant` 包；branch `master`，commit `89252a8f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 向量存储主类 | `QdrantVectorStore` 实现 langchain_core `VectorStore` 抽象，封装 Qdrant 集合的增删查 | `langchain_qdrant/qdrant.py:36` |
| 遗留兼容别名 | `Qdrant` 类（vectorstores.py 遗留模块，与 `QdrantVectorStore` 功能等价） | `langchain_qdrant/vectorstores.py:60` |
| 稠密向量写入 | `add_texts` 按 batch_size 分批，经 `_generate_batches` 组装点后 `client.upsert` | `qdrant.py:495` |
| 相似检索 | `similarity_search` / `similarity_search_with_score` 支持 filter/search_params/score_threshold/hybrid_fusion | `qdrant.py:520,551` |
| 向量直接检索 | `similarity_search_by_vector` 跳过嵌入，直接用向量检索 | `qdrant.py:699` |
| MMR 检索 | `max_marginal_relevance_search` 最大边际相关性去重 | `qdrant.py:728` |
| 删除与按 ID 查询 | `delete` 按 ids 删除；`get_by_ids` 按点 ID 取回 | `qdrant.py:856,877` |
| 工厂构造 | `from_texts` / `from_existing_collection` / `construct_instance` 多种实例化入口 | `qdrant.py:339,434,891` |
| 稀疏向量抽象 | `SparseEmbeddings` ABC 定义 `embed_documents`/`embed_query`，产出 `SparseVector` | `langchain_qdrant/sparse_embeddings.py:24` |
| FastEmbed 稀疏实现 | `FastEmbedSparse` 封装 fastembed 库做稀疏嵌入 | `langchain_qdrant/fastembed_sparse.py:11` |
| 检索模式枚举 | `RetrievalMode`（Dense/Sparse/Hybrid）控制稠密/稀疏/混合检索 | `qdrant.py:28` |
| 集合配置校验 | `_validate_collection_config` / `_validate_collection_for_dense/sparse` 启动时校验向量维度与距离函数 | `qdrant.py:1150,1179,1252` |
| 相关性评分归一化 | `_select_relevance_score_fn` 按距离函数（余弦/欧氏/点积）映射到 [0,1] | `qdrant.py:1004` |
| 同步回退装饰器 | `sync_call_fallback` 装饰器在同步方法中处理客户端同步/异步差异 | `langchain_qdrant/_utils.py:35`（vectorstores.py:35） |

对外导出面（`langchain_qdrant/__init__.py`）：`QdrantVectorStore`、`Qdrant`、`RetrievalMode`、`SparseEmbeddings`、`SparseVector`、`FastEmbedSparse`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `QdrantVectorStore(VectorStore)` | `qdrant.py:36` | 向量存储主类，持有 `client`（QdrantClient）、`collection_name`、`embedding`、`sparse_embedding`、`retrieval_mode` |
| `RetrievalMode(str, Enum)` | `qdrant.py:28` | 稠密/稀疏/混合检索模式枚举 |
| `SparseEmbeddings(ABC)` | `sparse_embeddings.py:24` | 稀疏嵌入抽象基类，定义 `embed_documents`/`embed_query` 契约；异步默认走 `run_in_executor` |
| `SparseVector(BaseModel)` | `sparse_embeddings.py:10` | 稀疏向量结构（indices + values，等长） |
| `FastEmbedSparse(SparseEmbeddings)` | `fastembed_sparse.py:11` | 基于 fastembed 的稀疏嵌入实现 |
| `_generate_batches` | `qdrant.py:1038` | 把 texts/metadatas/ids 按 batch_size 切成 (batch_ids, points) 批次 |
| `_build_payloads` / `_build_vectors` | `qdrant.py:1074,1098` | 把 Document metadata 转为 Qdrant payload dict；把嵌入结果转为 Qdrant 向量结构（稠密+稀疏） |
| `_document_from_point` | `qdrant.py:1023` | Qdrant SearchResult point → langchain `Document`（还原 page_content 与 metadata） |
| `_select_relevance_score_fn` | `qdrant.py:1004` | 按集合距离函数选择评分归一化函数 |
| `QdrantException` | `vectorstores.py:31` | 遗留异常类型 |

**依赖倒置**：`VectorStore` 抽象基类定义在 `langchain_core`，`QdrantVectorStore` 是其实现；`Embeddings` 抽象也在 langchain_core，由用户注入具体嵌入实现（如 `OpenAIEmbeddings`）。

## 3. 关键调用链

### 链 1：写入文本（add_texts）

1. 用户调用 `vector_store.add_texts(texts, metadatas, ids)`（`qdrant.py:495`）。
2. `add_texts` 遍历 `self._generate_batches(texts, metadatas, ids, batch_size=64)`（`:1038`）：内部调 `self._embedding.embed_documents(batch_texts)`（稠密嵌入），若启用稀疏则调 `self.sparse_embedding.embed_documents`；`_build_payloads`（`:1074`）把 metadata 包成 `{"metadata": ...}`，`_build_vectors`（`:1098`）组装稠密+稀疏向量。
3. 每批调用 `self.client.upsert(collection_name=self.collection_name, points=points)`（`:513`）写入 Qdrant 服务端。
4. 返回所有 added_ids 列表。

### 链 2：相似检索（similarity_search_with_score）

1. 用户调用 `vector_store.similarity_search(query, k=4)`（`:520`），内部转调 `similarity_search_with_score`（`:551`）。
2. `:569` 起构建 query_options：`collection_name`、`query_filter`、`search_params`、`limit=k`、`offset`、`score_threshold`、`hybrid_fusion`。
3. 查询向量经 `self._get_query_embedding(query)`（稠密）和/或稀疏嵌入得到，调 `self.client.query_points(...)` 或 `client.search(...)` 发到 Qdrant 服务端。
4. 命中点经 `_document_from_point`（`:1023`）还原为 `(Document, score)` 元组列表返回。

### 链 3：实例化与集合校验

1. 用户通过 `from_texts`（`:339`）或 `from_existing_collection`（`:434`）构造。
2. `__init__`（`:210`）保存 client/collection_name/embedding 等配置。
3. 首次操作时 `_validate_collection_config`（`:1150`）检查集合是否存在，`_validate_collection_for_dense`（`:1179`）校验稠密向量维度与距离函数与嵌入模型一致；稀疏模式走 `_validate_collection_for_sparse`（`:1252`）。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `collection_name` | 必填，目标 Qdrant 集合名 | `qdrant.py:210` |
| `embedding` | 稠密嵌入函数（用户注入，如 OpenAIEmbeddings） | `qdrant.py:210` |
| `sparse_embedding` | 可选稀疏嵌入函数 | `qdrant.py:210` |
| `retrieval_mode` | `RetrievalMode.Dense`（默认）；可设 Sparse/Hybrid | `qdrant.py` 枚举定义于 :28 |
| `batch_size` | `64`（add_texts 批次大小） | `qdrant.py:500` |
| `content_payload_key` / `metadata_payload_key` | 默认 `"page_content"` / `"metadata"` | payload 组装逻辑 |
| `vector_name` | 多向量集合中的向量名 | `qdrant.py` 配置字段 |
| `distance` | 集合距离函数（COSINE/EUCLID/DOT），创建集合时指定 | `qdrant.py:1179` 校验 |

## 5. 错误与重试语义

- **集合配置不匹配**：`_validate_collection_for_dense`（`:1179`）检测到向量维度或距离函数与嵌入模型不一致时抛 `QdrantVectorStoreError`（`:24`），阻止错误检索。
- **嵌入缺失**：`_require_embeddings`（`:302`）在需要嵌入但未配置 `embedding` 时抛 `ValueError`，指明需要哪个操作。
- **重试**：本层不实现重试；由 `qdrant-client` SDK 与 Qdrant 服务端负责。
- **删除**：`delete`（`:856`）透传 ids 到 `client.delete`，不做额外错误包装。

## 6. 并发细节

- **同步为主**：`add_texts`/`similarity_search` 均为同步方法；langchain_core `VectorStore` 基类提供默认异步包装（`aadd_texts`/`asimilarity_search`）走 `run_in_executor`。
- **批量写入**：`add_texts` 按 `batch_size=64` 分批 upsert，控制单次请求大小。
- **无共享可变状态**：实例字段即配置，`client` 在 `__init__` 注入后复用。
- **稀疏嵌入异步**：`SparseEmbeddings.aembed_documents`/`aembed_query`（sparse_embeddings.py）默认走 `run_in_executor` 包装同步实现。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `langchain_qdrant/qdrant.py`：`QdrantVectorStore` 主类、稠密+稀疏混合检索、集合校验
- `langchain_qdrant/vectorstores.py`：遗留 `Qdrant` 别名类（与主类功能等价）
- `langchain_qdrant/sparse_embeddings.py`：稀疏嵌入 ABC 与 SparseVector 结构
- `langchain_qdrant/fastembed_sparse.py`：FastEmbed 稀疏嵌入实现
- `langchain_qdrant/_utils.py`：同步回退装饰器

**Out-of-Scope（不在本仓库源码内）**

- Qdrant 向量数据库服务端（集合存储、HNSW 索引、向量检索计算）——外部服务
- `qdrant-client` Python SDK（gRPC/HTTP 客户端）——依赖项
- `fastembed` 库（稀疏嵌入模型推理）——依赖项
- 具体稠密嵌入模型（OpenAIEmbeddings 等）——由用户注入，见 partner-openai 叶子
- 相邻叶子：Chroma 向量库、Chat 模型提供商

## 8. 与相邻子系统交互

- **上游 → 本叶子**：RAG 链/检索链通过 `VectorStore` 抽象调用 `QdrantVectorStore`；用户注入 `Embeddings` 实现（如 OpenAIEmbeddings）做文本向量化。
- **本叶子 → 下游**：
  - `QdrantVectorStore.add_texts` → `qdrant-client` SDK `client.upsert` → Qdrant 服务端（外部）；
  - `QdrantVectorStore.similarity_search` → `qdrant-client` SDK `client.query_points` → Qdrant 服务端（外部）；
  - 稀疏嵌入经 `SparseEmbeddings` 抽象调用具体稀疏嵌入实现（FastEmbed，外部）。
- **同域交互**：依赖 `langchain_core.vectorstores.VectorStore` 与 `langchain_core.embeddings.Embeddings` 抽象；与其他 partner 包无直接代码依赖。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：核心能力缝是 **`VectorStore` 抽象实现**——langchain_core 定义 `VectorStore` 基类（`add_texts`/`similarity_search`/`delete` 等抽象方法），本包提供 Qdrant 具体实现。第二个能力缝是 **稀疏嵌入抽象**（`SparseEmbeddings` ABC），与 langchain_core 的稠密 `Embeddings` 抽象平行。
- **注册表与工厂**：`from_texts`/`from_existing_collection`/`construct_instance` 是工厂类方法；无字符串→类注册表。
- **可选依赖与懒加载**：`qdrant-client` 是硬依赖；`fastembed` 仅在使用 `FastEmbedSparse` 时需要；嵌入模型由用户注入，本包不绑定具体嵌入实现。
- **配置驱动**：`RetrievalMode` 枚举（Dense/Sparse/Hybrid）决定查询时走稠密向量、稀疏向量还是两者混合——同一 `similarity_search_with_score` 代码路径按配置分派到不同查询构建逻辑。
- **双轨实现**：`QdrantVectorStore`（qdrant.py）与遗留 `Qdrant`（vectorstores.py）是同一能力的双轨，新代码应使用 `QdrantVectorStore`。
- **外部边界**：Qdrant 服务端、`qdrant-client` SDK、`fastembed` 均标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Qdrant 集成架构图 | `partner-qdrant-architecture.html` | architecture | **showcase** |
| 写入与检索数据流图 | `partner-qdrant-dataflow.html` | dataflow | **showcase** |

**说明**：本叶子不补 sequence 图——核心写入/检索链路是单向管道（文本→嵌入→upsert/query→结果），无多方往返消息交互，与 dataflow 图信息重复，按资源节省原则省略。不补 lifecycle 图——Qdrant 点/集合无明确的多状态机语义。两图均一次通过 showcase 校验。JSON IR 源文件位于 `json/` 目录。
