# 回调、缓存、Hub 与序列化基础设施（classic-callbacks-infra）

> 本文是 `classic` 域下的叶子子系统文档。域级总览见 `../classic.md`，本文只展开经典包的
> **回调系统、LLM 缓存、LangSmith Hub 拉取、序列化加载与全局状态**，不展开业务子系统。
>
> 源码基准：`langchain_classic` 1.0.8，分支 `master`，commit `4492ad7a8`。

## 1. 功能清单

源码位于 `callbacks/`（45 文件）、`cache.py`、`hub.py`、`smith/`（7 文件）、`load/`（4 文件）、
`adapters/`（2 文件）、`schema/`（43 文件）、`globals.py`、`env.py`、`_api/`（5 文件）。

| 能力 | 说明 | 源码路径 |
| --- | --- | --- |
| 回调基类重导出 | `BaseCallbackHandler` 等 mixin 重导出 core | `callbacks/base.py` |
| 回调管理器 | 编排一次运行的回调分发 | `callbacks/manager.py` |
| 集成回调 | ~25 种平台回调（wandb/mlflow/streamlit/…） | `callbacks/*_callback.py` |
| 流式回调 | 流式 stdout/aiter 回调 | `callbacks/streaming_*.py` |
| 追踪器 | LangSmith 等 tracer | `callbacks/tracers/` |
| LLM 缓存 | 缓存实现转发 community | `cache.py` |
| LangSmith Hub | `pull`/`push` 拉取/推送序列化对象 | `hub.py:19`、`:69` |
| 序列化 | dump/load/serializable | `load/dump.py`、`load/load.py`、`load/serializable.py` |
| LangSmith 集成 | smith 包 | `smith/` |
| 适配器 | OpenAI 适配 | `adapters/openai.py` |
| schema | 消息/AIMessage 等 schema | `schema/` |
| 全局状态 | verbose/调试开关 | `globals.py` |
| 弃用基础设施 | `create_importer`/deprecation | `_api/` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
| --- | --- | --- |
| `BaseCallbackHandler` / 各 ManagerMixin | `callbacks/base.py`（重导出 core） | 回调钩子契约：on_chain_start/end、on_llm_start 等 |
| `CallbackManager` / `AsyncCallbackManager` | `callbacks/manager.py` | 合并运行时与构造期回调、派生 run manager |
| `hub.pull(owner_repo_commit)` | `hub.py:69` | 从 LangSmith Hub 拉取并反序列化 prompt/对象 |
| `hub.push` | `hub.py:19` | 把对象推送到 Hub |
| `dumpd`/`load` | `load/dump.py`、`load/load.py` | 对象↔可序列化 dict（LCEL 兼容） |
| `Serializable` | `load/serializable.py` | 可序列化基类（`is_lc_serializable`） |
| `create_importer` | `_api/module_import.py` | 延迟导入桩工厂 |

## 3. 关键调用链

**一次运行的回调分发**（与 classic-chains 的 `invoke` 呼应）：
`CallbackManager.configure(...)` 合并回调 → `on_chain_start` → 运行中各节点
`on_llm_start`/`on_tool_start` → `on_chain_end`/`on_chain_error`。

**Hub 拉取**（`hub.pull`，`hub.py:69`）：
按 `owner/prompt:commit` 调 LangSmith Hub API → 拿到 manifest → 经 `load()` 反序列化为
LangChain 对象。**安全提示**：`pull` 文档明确警告 manifest 是不可信输入，建议 pin commit、
审计后再反序列化。

**序列化**：`Serializable.dump()` 生成含 `_type` 的 dict；`load()` 按 `_type` 路由重建对象，
与 `load_chain`/`load_agent` 共用注册表范式。

## 4. 配置项

| 配置 / 参数 | 默认 / 行为 | 位置 |
| --- | --- | --- |
| `verbose` 全局开关 | 控制详细日志 | `globals.py`、`env.py` |
| Hub `api_url`/`api_key` | 默认托管 LangSmith 或本地 | `hub.py:69` |
| `include_model` | 拉取时是否含模型配置 | `hub.py:70` |
| 缓存类型 | 经 `set_llm_cache` 全局设置 | `cache.py` |

## 5. 错误与重试语义

- **Hub 拉取**：网络/鉴权错误向上抛；manifest 反序列化安全风险由文档警告，不自动屏蔽。
- **回调异常**：默认不阻断主流程（日志类回调容错），关键回调由 manager 控制。
- **缓存**：缓存读写失败通常降级为不命中、重新推理。
- **弃用**：`create_importer` 触发 `LangChainDeprecationWarning`。

## 6. 并发细节

- 回调管理器区分同步 `CallbackManager` 与异步 `AsyncCallbackManager`，分别服务同步/异步执行。
- 无自建 worker 线程；流式回调通过生成器/异步迭代器推送 token。
- Hub 拉取为单次同步/HTTP 调用，无并发模型。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- 回调基类重导出、集成回调桩、Hub 客户端、序列化工具、全局状态、弃用设施。

**Out-of-Scope（不在本仓库源码内）**
- 回调管理器核心与消息 schema 在 `langchain-core`。
- LangSmith Hub 为**外部托管服务**（`smith.langchain.com`）。
- 各回调集成平台（wandb/mlflow/streamlit 等）为外部服务。
- LLM 缓存后端（Redis/SQL/向量库）实现与外部存储不在本仓库源码内。

## 8. 与相邻子系统交互

- 上游：所有 Chain/Agent/LLM（各业务叶子）→ 经回调管理器分发钩子。
- 本叶子 → 下游：
  - 回调 → core 基类 + 各外部观测平台。
  - `hub.pull` → LangSmith Hub → `load()` 反序列化 Chain/Prompt。
  - `set_llm_cache` → 缓存后端。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：回调钩子是"可观测性能力缝"——所有运行节点经统一 hook 点上报；
  缓存是"横切关注点"能力缝；Hub 是"远程对象拉取"能力缝。
- **注册表与工厂**：`load()` 按 `_type` 路由，与 `load_chain`/`load_agent` 同范式。
- **可选依赖与懒加载**：各平台回调按第三方 SDK 延迟 import。
- **与 core/v1 的关系**：回调基类、消息 schema 已在 core；本包提供集成桩、Hub 客户端与
  全局状态。Hub 拉取是经典包特有的跨包机制。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
| --- | --- | --- | --- |
| 回调/缓存/Hub 基础设施架构图 | `classic-callbacks-infra-architecture.html` | architecture | showcase |

降档披露：组件边界清晰，保持 `showcase`。JSON IR 位于 `json/`。
