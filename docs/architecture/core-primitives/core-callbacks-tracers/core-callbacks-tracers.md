# 回调与追踪（core-callbacks-tracers）

> 本文是 `core-primitives` 域下的叶子子系统文档。域级总览见 `../core-primitives.md`。
> 本文展开「横切关注点（追踪/日志/流式）的回调注入点」，不重复展开 Runnable 执行链（见 `../core-runnables/core-runnables.md`）。
>
> 源码基准：`langchain-core` master，commit `4492ad7a804e94bdcc89bf148cfc8efd5e7d6ef7`，源码位于 `libs/core/langchain_core/callbacks/` + `tracers/`。

## 1. 功能清单

| 能力 | 说明 | 源码路径 |
|------|------|----------|
| 回调 Mixin 族 | `LLMManagerMixin`/`ChainManagerMixin`/`ToolManagerMixin`/`RetrieverManagerMixin`/`RunManagerMixin`/`CallbackManagerMixin`：定义 `on_*_start/end/error` 钩子 | `callbacks/base.py:62`/`169`/`241`/`24`/`435`/`279` |
| `BaseCallbackHandler` | 回调处理器基类，含 `ignore_llm`/`ignore_chain` 等过滤开关 | `callbacks/base.py:496` |
| `BaseRunManager` | 单次 run 的管理器基类 | `callbacks/manager.py:490` |
| 同步 RunManager | `RunManager`/`ParentRunManager`：同步 run 树管理，`get_child` 派生子 run | `manager.py:546`/`599` |
| 异步 RunManager | `AsyncRunManager`/`AsyncParentRunManager` | `manager.py:621`/`683` |
| 类型化 Run 管理器 | `CallbackManagerForLLMRun`/`ForChainRun`/`ForToolRun`：按 run 类型混入对应 Mixin | `manager.py:705`/`928` |
| 回调管理器 | `CallbackManager`/`AsyncCallbackManager`：聚合多个 handler，`on_chain_start` 等广播到全部 | `callbacks/manager.py` |
| 执行期取管理器 | `get_callback_manager_for_config`：从 RunnableConfig 构建管理器 | `runnables/config.py:563` |
| `BaseTracer` | 把 run 事件持久化为 `Run` 对象的 ABC；`_persist_run`/`_start_trace`/`_end_trace` | `tracers/base.py:33` |
| `AsyncBaseTracer` | 异步版追踪器 | `tracers/base.py:551` |
| Run 数据模型 | `Run`/`RunTree`：单次执行的结构化记录（inputs/outputs/error/child_runs） | `tracers/core.py`、`tracers/schemas.py` |
| 流式日志 | `astream_log` 经 `LogStreamCallbackHandler`/`run_collector` 收集中间步骤 | `tracers/log_stream.py`、`run_collector.py` |
| 事件流 | `astream_events` v2 经 `event_stream.py` 产出标准化事件 | `tracers/event_stream.py` |
| 内置 handler | `StdOutCallbackHandler`/`FileCallbackHandler`/`UsageMetadataCallbackHandler` | `callbacks/stdout.py`/`file.py`/`usage.py` |
| 监听器 | `with_listeners`/`with_alisteners` 注册 `on_start/on_end` 钩子 | `tracers/root_listeners.py` |

## 2. 核心类型与接口清单

| 类型 / 函数 | 位置 | 职责 |
|-------------|------|------|
| `BaseCallbackHandler` | `callbacks/base.py:496` | 回调处理器契约；用户/框架实现它接收 `on_*` 事件 |
| `CallbackManager` / `AsyncCallbackManager` | `callbacks/manager.py` | 聚合多个 handler 的广播器；`on_chain_start` 同时通知所有 handler |
| `ParentRunManager` / `AsyncParentRunManager` | `manager.py:599`/`683` | 一次 run 的句柄，`get_child(tag)` 派生子 run，形成 run 树 |
| `BaseTracer` | `tracers/base.py:33` | 同时是 `BaseCallbackHandler` 子类：把事件转成 `Run` 并 `_persist_run` |
| `Run` | `tracers/core.py`/`schemas.py` | 一次执行的结构化记录：run_type/inputs/outputs/error/child_runs/start_time |
| `get_callback_manager_for_config` | `runnables/config.py:563` | 从 config 中的 callbacks 列表构建管理器（同步/异步） |

## 3. 关键调用链

**调用链一：Runnable 执行触发回调广播（`_call_with_config`，见 core-runnables）**

1. `ensure_config` → `get_callback_manager_for_config(config)` 构建 `CallbackManager`。
2. `callback_manager.on_chain_start(serialized, input, name=...)`：管理器向所有注册的 handler 广播开始事件，并返回 `run_manager`。
3. 子步经 `run_manager.get_child("seq:step:1")` 派生子 `CallbackManager`，形成 run 树。
4. 正常结束 `run_manager.on_chain_end(output)`；异常 `run_manager.on_chain_error(e)`。
5. 每个 handler 决定是否关心该事件（`ignore_chain`/`ignore_llm` 过滤）。

**调用链二：Tracer 把事件持久化为 Run（`BaseTracer`，`tracers/base.py:33`）**

1. `BaseTracer` 本身是 `BaseCallbackHandler`，实现 `on_llm_start`/`on_chain_end` 等。
2. 每个事件构造/更新一个 `Run` 对象（含 inputs/outputs/error/child_runs）。
3. `_persist_run(run)` 把 Run 持久化——内置 stdout/file 直接写；LangSmith tracer（外部，不在本仓库源码内）上传到云。

**调用链三：流式事件（`astream_events`/`astream_log`）**

1. `astream_events`（`runnables/base.py:1368`）经 `event_stream.py` 把 run 树事件转成标准化 `StreamEvent` 异步流。
2. `astream_log` 经 `LogStreamCallbackHandler` 收集中间步骤 JSON patch 流。

## 4. 配置项

| 配置项 | 默认 / 行为 | 位置 |
|--------|-------------|------|
| `RunnableConfig.callbacks` | 本次调用附加的 handler/manager 列表 | `runnables/config.py:92` |
| `RunnableConfig.tags`/`metadata` | 随 run 传播的过滤标签与元数据 | `config.py:80`/`86` |
| `BaseCallbackHandler.ignore_llm`/`ignore_chain`/`ignore_retry` | handler 级事件过滤开关 | `callbacks/base.py:513-523` |
| `set_debug`/`set_tracing` 全局开关 | 全局开启调试/追踪 | `globals.py`（见 core-serialization-cache-infra） |

## 5. 错误与重试语义

- **回调异常隔离**：回调是横切关注点，handler 抛错不应中断主业务流；run_manager 在 `on_chain_error` 中记录，主异常仍上抛。
- **重试事件**：`on_retry`（`callbacks/base.py:455`）由 `with_retry` 的 tenacity 钩子触发，记录重试次数与退避。
- **追踪异步化**：`AsyncBaseTracer` 的持久化是 fire-and-forget，不阻塞主执行；外部 LangSmith 上传失败不影响业务。

## 6. 并发细节

- **同步/异步双轨**：`CallbackManager`/`RunManager`（同步）与 `AsyncCallbackManager`/`AsyncParentRunManager`（异步）成对存在；`AsyncRunManager.get_sync()` 提供桥接。
- **run 树并发**：`RunnableParallel` 并发分支时，每个分支独立 `get_child` 派生子 run，run 树节点并发更新；`Run` 对象用 run_id(UUID) 标识，无共享可变状态。
- **contextvars 传播**：当前 run_manager 经 `set_config_context` 在 contextvars 中传递，子线程/协程自动继承（见 core-runnables）。
- **流式背压**：`event_stream`/`log_stream` 是 async 生成器，背压自然承担。

## 7. 系统边界

**In-Scope（本仓库源码内）**
- `callbacks/` 全部 Mixin/Handler/Manager；`tracers/` 全部 Tracer/Run 模型/流式收集。

**Out-of-Scope（不在本仓库源码内）**
- LangSmith 云后端的上传/UI 不在本仓库源码内；本叶子只定义 `BaseTracer` 契约与 Run 结构。
- 具体第三方观测后端（OpenTelemetry 等）的集成在外。

## 8. 与相邻子系统交互

- **上游（`core-runnables`）**：每个 Runnable 执行入口经 `get_callback_manager_for_config` 创建 run_manager，是横切注入点。
- **本叶子 → 模型（`core-language-models`）**：`on_llm_new_token` 把流式 token 推给 handler；`on_chat_model_start` 传 messages。
- **本叶子 → 工具（`core-tools`）**：`on_tool_start/end/error` 包裹工具执行。
- **本叶子 → 序列化（`core-serialization-cache-infra`）**：`Run`/`tracer` 与 `Serializable` 体系协作记录结构。

## 9. 语言专项适配口径（纯 Python 框架）

- **能力缝归组**：本叶子是「可插拔钩子链」能力缝的典型——`BaseCallbackHandler` ABC 定义 `on_*` 钩子，`CallbackManager` 是事件广播器，`BaseTracer` 是观察者实现。
- **观察者模式**：执行核心（Runnable）不感知 handler 细节，只经 manager 广播；handler 可随时增删，是依赖倒置。
- **双轨（同步/异步）**：每个 Manager/Tracer 都有 sync/async 成对实现。
- **图类型侧重**：用 dataflow 表达「执行事件 → 管理器广播 → handler/tracer 持久化」管道；不补 lifecycle。

## 10. 图表清单与质量

| 图 | 文件 | 类型 | archify 质量档 |
|----|------|------|----------------|
| 回调广播与追踪管道数据流图 | `core-callbacks-tracers-dataflow.html` | dataflow | standard（showcase 对同一扇出源多流标签间距校验严格，降 standard） |

JSON IR 源文件位于 `json/core-callbacks-tracers-dataflow.json`。本叶子以事件管道为主，故用 dataflow 而非 architecture。
