# partner-search-tools 叶子子系统分析

> 本文是 `partners` 域下的叶子子系统文档。域级总览见 `../partners.md`。
> 本文只展开 Exa 搜索 API 集成（`langchain-exa`）的职责边界，不重复展开其他相邻叶子。
>
> 源码基准：`libs/partners/exa/`，`langchain-exa` 包；branch `master`，commit `89252a8f`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|---|---|---|
| Exa 搜索工具 | `ExaSearchResults`：根据文本 query 调 Exa 搜索 API，返回 JSON 结果 | `langchain_exa/tools.py:21` |
| 相似页面工具 | `ExaFindSimilarResults`：根据 URL 找相似页面 | `tools.py:164` |
| Exa 检索器 | `ExaSearchRetriever`：继承 `BaseRetriever`，搜索结果转为 langchain Document | `langchain_exa/retrievers.py:36` |
| 客户端初始化 | `initialize_client`：解析 `EXA_API_KEY` / `exa_base_url`，构建 `Exa` SDK 客户端 | `langchain_exa/_utilities.py:7` |
| 结果元数据提取 | `_get_metadata`：把 Exa 结果对象转为 Document metadata dict | `retrievers.py:20` |

对外导出面（`langchain_exa/__init__.py`）：`ExaSearchResults`、`ExaFindSimilarResults`、`ExaSearchRetriever`、`HighlightsContentsOptions`、`TextContentsOptions`。

**注意**：`ExaGetContents` 在当前源码中未单独成类——内容获取能力通过 `ExaSearchResults` 的 `text_contents_options`/`highlights`/`summary` 参数以及 `search_and_contents` 方法实现，而非独立工具类。如实披露此差异。

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|---|---|---|
| `ExaSearchResults(BaseTool)` | `tools.py:21` | Exa 搜索工具，`_run` 调 `client.search_and_contents`，输入 query + 过滤参数，返回结果 dict 列表 |
| `ExaFindSimilarResults(BaseTool)` | `tools.py:164` | 相似页面工具，`_run` 调 `client.find_similar_and_contents`，输入 URL |
| `ExaSearchRetriever(BaseRetriever)` | `retrievers.py:36` | 检索器，`_get_relevant_documents` 调 `search_and_contents`，结果转为 `Document` 列表 |
| `initialize_client` | `_utilities.py:7` | 从 env `EXA_API_KEY` 或参数解析 key，构建 `Exa` SDK 客户端 |
| `_get_metadata` | `retrievers.py:20` | Exa Result 对象 → metadata dict（title/url/score/author/highlights/summary） |

**依赖倒置**：工具继承 `langchain_core.tools.BaseTool`，检索器继承 `langchain_core.retrievers.BaseRetriever`——接口在消费方 `langchain_core`，实现在本包。

## 3. 关键调用链

### 链 1：Exa 搜索工具调用

1. Agent 调用 `ExaSearchResults._run(query="...", num_results=10, ...)`（`tools.py:101`）。
2. `_run` 直接调 `self.client.search_and_contents(query, num_results=..., text=..., highlights=..., include_domains=..., exclude_domains=..., start/end_crawl_date=..., start/end_published_date=..., use_autoprompt=..., livecrawl=..., summary=..., type=...)`（`:144`）。
3. `Exa` SDK 客户端发 HTTPS 请求到 Exa API 服务端。
4. 返回 `SearchResponse` 对象（含 `results` 列表，每项有 url/title/text/highlights/summary 等）。
5. 异常路径：`except Exception as e: return repr(e)`（`:160-161`）——错误转为字符串返回，不抛异常。

### 链 2：Exa 检索器

1. 用户调用 `retriever.invoke("query")`，`BaseRetriever` 框架层调用 `_get_relevant_documents`（`retrievers.py:79`）。
2. 内部调 `self.client.search_and_contents(query, num_results=self.k, ...)`（`:82`）。
3. 遍历 `response.results`，每项经 `_get_metadata(result)`（`:20`）提取 metadata，文本作为 `page_content`。
4. 返回 `list[Document]`。

### 链 3：客户端初始化

1. `ExaSearchResults` / `ExaFindSimilarResults` / `ExaSearchRetriever` 均用 `@model_validator(mode="before")` 调 `initialize_client(values)`（`_utilities.py:7`）。
2. 解析顺序：`values["exa_api_key"]` → 环境变量 `EXA_API_KEY` → 空字符串。
3. 若提供 `exa_base_url` 则传入；构建 `Exa(api_key=..., base_url=...)` 实例存入 `values["client"]`。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|---|---|---|
| `exa_api_key` | 从 `EXA_API_KEY` 环境变量读取 | `tools.py:93`、`_utilities.py:9` |
| `exa_base_url` | `None`（用 Exa 默认端点） | `tools.py:175`、`_utilities.py:14` |
| `num_results` | `10`（工具 `_run` 参数）；检索器用 `k=10` | `tools.py:104`、`retrievers.py:39` |
| `text_contents_options` | 页面内容获取选项（True 或 dict） | `tools.py:105` |
| `highlights` | 高亮摘要选项 | `tools.py:109` |
| `include_domains` / `exclude_domains` | 域名过滤 | `tools.py:110-111` |
| `start/end_crawl_date` | 爬取日期范围 | `tools.py:112-113` |
| `start/end_published_date` | 发布日期范围 | `tools.py:114-115` |
| `use_autoprompt` | 自动提示优化 | `tools.py:116` |
| `livecrawl` | `"always"/"fallback"/"never"` | `tools.py:117` |
| `summary` | 摘要选项 | `tools.py:118` |
| `type` | `"auto"/"deep"/"fast"` | `tools.py:119` |
| `category` | 相似页面分类过滤（FindSimilar 专用） | `tools.py:199` |
| `exclude_source_domain` | 是否排除源域名（FindSimilar 专用） | `tools.py:198` |

## 5. 错误与重试语义

- **错误处理**：`_run` 方法用 `try/except Exception` 包裹 SDK 调用，失败时 `return repr(e)`（`tools.py:160-161`）——错误转为字符串返回给调用方，不抛异常中断 agent 循环。这是工具类的容错设计。
- **重试**：本层不实现重试；由 `exa_py` SDK 或用户上层负责。
- **API key 缺失**：`initialize_client` 中 key 为空字符串时仍构建客户端（`_utilities.py:9-16`），实际 API 调用时由 Exa 服务端返回 401。

## 6. 并发细节

- **同步工具**：`ExaSearchResults._run` 与 `ExaFindSimilarResults._run` 均为同步方法，无 `_arun` 异步实现（基类 `BaseTool` 提供默认异步包装）。
- **检索器**：`_get_relevant_documents` 同步；`BaseRetriever` 提供默认 `_aget_relevant_documents` 包装。
- **无共享可变状态**：`client` 是 `Exa` SDK 客户端实例，在 `validate_environment` 一次性构建。

## 7. 系统边界

**In-Scope（本仓库源码内）**

- `langchain_exa/tools.py`：两个搜索工具
- `langchain_exa/retrievers.py`：检索器
- `langchain_exa/_utilities.py`：客户端初始化

**Out-of-Scope（不在本仓库源码内）**

- Exa 搜索 API 服务端——外部服务
- `exa_py` Python SDK（依赖项）
- 相邻叶子：Chat 模型、Embeddings、向量存储

## 8. 与相邻子系统交互

- **上游 → 本叶子**：Agent（如 LangGraph `create_react_agent`）通过 `BaseTool` 接口调用 `ExaSearchResults`；检索链通过 `BaseRetriever` 接口调用 `ExaSearchRetriever`。
- **本叶子 → 下游**：
  - 工具/检索器 → `exa_py.Exa` SDK → Exa API 服务端（外部）。
- **同域交互**：依赖 `langchain_core.tools.BaseTool` / `langchain_core.retrievers.BaseRetriever` 抽象；与其他 partner 包无直接代码依赖。

## 9. 语言专项适配口径（纯 Python 框架库）

- **能力缝归组**：本叶子的能力缝是 **`BaseTool` / `BaseRetriever` 接口实现**——把 Exa 搜索 API 包装为 langchain 工具（供 agent 调用）与检索器（供 RAG 链调用）。两个工具类结构高度同构（同样的 `validate_environment` + `_run` 模板），仅 SDK 调用方法不同（`search_and_contents` vs `find_similar_and_contents`）。
- **注册表与工厂**：`initialize_client` 是工厂函数（从 env/参数构建 SDK 客户端）；无字符串→类注册表。
- **可选依赖**：`exa_py` 是硬依赖；API key 通过环境变量注入。
- **配置驱动**：搜索行为（内容获取/高亮/摘要/域名过滤/日期范围/livecrawl/搜索类型）全部通过 `_run` 方法参数传递，而非配置对象。
- **外部边界**：Exa API 服务端、`exa_py` SDK 标注"不在本仓库源码内"。
- **如实披露**：`ExaGetContents` 在当前源码中不是独立类，内容获取能力通过 `ExaSearchResults` 的参数实现。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|---|---|---|---|
| Exa 搜索工具架构图 | `partner-search-tools-architecture.html` | architecture | **showcase** |
| ExaSearchResults 调用时序图 | `partner-search-tools-sequence.html` | sequence | **showcase** |

**第三轮刷新**：补全 `meta.output` 字段后重渲染，架构图由第二轮的 standard 提升为 showcase（上游 CLI 升级后布局校验重新通过），时序图保持 showcase。关键行号已按 HEAD `89252a8f` 复核（`ExaSearchResults` @ tools.py:21、`_run` @ :101、`search_and_contents` @ :144、`ExaSearchRetriever` @ retrievers.py:36、`initialize_client` @ _utilities.py:7），与第二轮一致。JSON IR 源文件位于 `json/` 目录。
