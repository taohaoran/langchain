# partner-chroma 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文只展开 Chroma 向量数据库集成（`langchain-chroma`）的职责边界，不重复展开 Qdrant 等其他向量库、
> Chat 模型提供商、Exa 搜索等相邻叶子。
>
> 源码基准：`libs/partners/chroma/`，`langchain-chroma` 包；branch `master`，commit `89252a8f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| 向量存储主类 | `Chroma` 实现 langchain_core `VectorStore`，封装 Chroma 集合增删查 | `langchain_chroma/vectorstores.py:155` |
| 文本写入 | `add_texts` 嵌入文本后经 `collection.add` 写入，自动分配 UUID | `vectorstores.py:597` |
| 图像写入 | `add_images` 把图像 URI base64 编码后写入多模态集合 | `vectorstores.py:509` |
| 相似检索 | `similarity_search` / `similarity_search_with_score` | `vectorstores.py:730,817` |
| 向量直接检索 | `similarity_search_by_vector` 跳过嵌入直接查 | `vectorstores.py:756` |
| 图像检索 | `similarity_search_by_image` 以图搜图 | `vectorstores.py:945` |
| 混合检索 | `hybrid_search` 组合向量与关键词搜索 | `vectorstores.py:684` |
| MMR 检索 | `max_marginal_relevance_search` 最大边际相关性 | `vectorstores.py:1075` |
| 文档更新/删除 | `update_document`/`update_documents`/`delete`/`delete_collection` | `vectorstores.py:1214,1223,1452,1124` |
| 按 ID 查询 | `get` / `get_by_ids` 按点 ID 取回 Document | `vectorstores.py:1137,1178` |
| 工厂构造 | `from_texts` / `from_documents` | `vectorstores.py:1273,1375` |
| 集合分叉 | `fork(new_name)` 复制当前集合为新集合 | `vectorstores.py:492` |
| MMR 工具函数 | `maximal_marginal_relevance` 纯函数实现 MMR 算法 | `vectorstores.py:109` |
| 结果转换 | `_results_to_docs` / `_results_to_docs_and_scores` / `_results_to_docs_and_vectors` | `vectorstores.py:35,39,58` |

对外导出面（`langchain_chroma/__init__.py`）：`Chroma`。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `Chroma(VectorStore)` | `vectorstores.py:155` | 向量存储主类，持有 `client`、`collection_name`、`_embedding_function`、`persist_directory` |
| `_collection` property | `vectorstores.py:432` | 懒加载 `chromadb.Collection` 句柄（`__ensure_collection` 确保集合存在） |
| `__query_collection` | `vectorstores.py:450` | 内部查询方法，封装 embedding → collection.query → 结果转换 |
| `_results_to_docs` | `vectorstores.py:35` | Chroma raw results → `list[Document]` |
| `_results_to_docs_and_scores` | `vectorstores.py:39` | results → `list[(Document, score)]` |
| `maximal_marginal_relevance` | `vectorstores.py:109` | MMR 纯函数：在相关性与多样性间权衡选文档 |
| `cosine_similarity` | `vectorstores.py:79` | 余弦相似度矩阵计算（numpy） |
| `encode_image` | `vectorstores.py:487` | 静态方法：图像 URI → base64 data URL |

**依赖倒置**：`VectorStore` 抽象在 `langchain_core`，`Chroma` 是其实现；嵌入函数由用户注入。

## 3. 关键调用链

### 链 1：相似检索

1. 用户调用 `chroma.similarity_search(query, k=4)`（`vectorstores.py:730`）。
2. 内部转调 `__query_collection`（`:450`）：先调 `self._embedding_function.embed_query(query)` 得到查询向量。
3. 调 `self._collection.query(query_embeddings=[query_embedding], n_results=k, where=where, ...)`（chromadb SDK）。
4. 返回的 raw results 经 `_results_to_docs`（`:35`）转为 `list[Document]`。
5. `similarity_search_with_score`（`:817`）走 `_results_to_docs_and_scores`（`:39`）附带距离分数。

### 链 2：写入文本

1. `add_texts`（`:597`）：若无 ids 则用 `uuid.uuid4()` 自动分配（`:622`）。
2. 若配置了嵌入函数，调 `self._embedding_function.embed_documents(texts)` 得到嵌入向量。
3. metadatas 长度不足时补空 dict（`:635`）。
4. 调 `self._collection.add(ids=ids, embeddings=embeddings, metadatas=metadatas, documents=texts)` 写入 Chroma。

### 链 3：MMR 检索

1. `max_marginal_relevance_search`（`:1075`）先取回较多候选（`fetch_k` 个）。
2. 候选经 `maximal_marginal_relevance`（`:109`）纯函数：在余弦相似度矩阵上贪心选择，平衡与 query 的相关性与候选间的多样性。
3. 返回 top-k 文档。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `collection_name` | `"langchain"` | `vectorstores.py:302` |
| `embedding_function` | 嵌入函数（用户注入） | `vectorstores.py:302` |
| `persist_directory` | Chroma 本地持久化目录 | `vectorstores.py:302` |
| `client` | 外部注入的 chromadb 客户端 | `vectorstores.py:302` |
| `collection_metadata` | 集合元数据（如 HNSW 配置） | `vectorstores.py:302` |
| `tags` | 可观测性标签 | `vectorstores.py:302` |

## 5. 错误与重试语义

- **元数据校验**：`add_texts`（`:597`）检测 metadatas 长度与 texts 不匹配时补空 dict 而非报错；metadata 类型错误时抛 `ValueError`。
- **集合不存在**：`__ensure_collection`（`:422`）在首次访问时创建集合。
- **重试**：本层不实现重试；由 `chromadb` SDK 与 Chroma 服务端负责。
- **删除**：`delete`（`:1452`）透传 ids 到 `collection.delete`。

## 6. 并发细节

- **同步为主**：`add_texts`/`similarity_search` 同步；基类提供异步包装。
- **集合句柄懒加载**：`_collection` property 首次访问时经 `__ensure_collection` 创建。
- **无共享可变状态**：实例字段即配置。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `langchain_chroma/vectorstores.py`：`Chroma` 主类、文本/图像/混合检索、MMR、结果转换

**Out-of-Scope（不在本仓库源码内）**

- Chroma 向量数据库服务端（集合存储、索引、检索计算）——外部服务
- `chromadb` Python SDK——依赖项
- 嵌入模型（OpenAIEmbeddings 等）——用户注入
- 相邻叶子：Qdrant、Chat 模型提供商

## 8. 与相邻子系统交互

- **上游 → 本叶子**：RAG 链通过 `VectorStore` 抽象调用 `Chroma`；用户注入 `Embeddings` 实现。
- **本叶子 → 下游**：`Chroma` → `chromadb.Collection` → Chroma 服务端（外部）。
- **同域交互**：依赖 `langchain_core.vectorstores.VectorStore` 与 `Embeddings` 抽象；与其他 partner 包无直接代码依赖。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：核心能力缝是 `VectorStore` 抽象的 Chroma 实现；本包单文件（vectorstores.py）承载全部适配逻辑。
- **注册表与工厂**：`from_texts`/`from_documents` 工厂类方法；无字符串→类注册表。
- **可选依赖**：`chromadb` 是硬依赖；嵌入函数由用户注入。
- **配置驱动**：通过 `chromadb.Settings`/collection_metadata 配置 Chroma 行为（持久化、HNSW 参数），同一 `Chroma` 类按配置连接本地或远端 Chroma。
- **外部边界**：Chroma 服务端、`chromadb` SDK 标注"不在本仓库源码内"。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Chroma 集成架构图 | `partner-chroma-architecture.html` | architecture | **showcase** |
| 相似检索时序图 | `partner-chroma-sequence.html` | sequence | **showcase** |

**说明**：本叶子不补 dataflow 图——写入/检索管道与 sequence 图表达同一链路（嵌入→查询→结果），信息重复；不补 lifecycle 图——Chroma 集合/点无明确多状态机语义。两图均一次通过 showcase 校验。JSON IR 源文件位于 `json/` 目录。
